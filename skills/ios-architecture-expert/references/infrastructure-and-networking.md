# Infrastructure — networking, connectivity, persistence details, logging

Use this when:
- Someone wants to "check if there's internet" before making a request
- You are configuring a `URLSession` and deciding what belongs on the configuration
- You need to enumerate the states an infrastructure API can actually return
- You are storing binary data, or optimizing a store that is doing too much I/O
- You need to add logging or a graceful-degradation fallback without polluting every module

Skip this file if:
- You need layer boundaries or where a type belongs. Use `architecture-layers.md`.
- You need the Scheduler / isolation rationale. Use `concurrency-at-boundaries.md`.
- You need to decide whether you need a persistence layer at all. That's in
  `architecture-layers.md` (URLCache and YAGNI).

Jump to:
- [Never pre-flight a connectivity check](#never-pre-flight-a-connectivity-check)
- [Configuring the session instead](#configuring-the-session-instead)
- [URLSession retains its delegate](#urlsession-retains-its-delegate)
- [Enumerate representable states](#enumerate-representable-states)
- [Persistence details](#persistence-details)
- [Optimize in the infrastructure, not the service layer](#optimize-in-the-infrastructure-not-the-service-layer)
- [Logging and graceful degradation](#logging-and-graceful-degradation)
- [Auth as a cross-cutting concern](#auth-as-a-cross-cutting-concern--the-decorator-youll-actually-write)

---

## Never pre-flight a connectivity check

This is the single most common infrastructure mistake in iOS apps, and it has an unambiguous
answer from Apple:

> **"Always attempt to make a connection. Do not attempt to guess whether network service is
> available, and do not cache that determination."**

Using `SCNetworkReachability` or `NWPathMonitor` to decide *whether* to make a request is not
reliable, and it produces the worst possible user experience: a device that is plainly online while
the app insists it is not. Determining current reachability is riddled with race conditions — the
network can change between the check and the request.

**Reachability is a diagnostic, not a gate.** Apple's own framing: the API is not intended as a
preflight mechanism for determining connectivity; if a connection *fails*, consult it to help
diagnose the cause. Legitimate uses:

- Auto-retry once connectivity returns
- Show a hint after a failure — "Looks like you're offline"

> The rule of thumb: **set the appropriate constraints, attempt the request, and handle the result.**

Architecturally, this is also why the feed loader has no connectivity concept at all. The
remote-with-cache-fallback composition *is* the offline strategy: attempt remote, fall back to
cache. No flag, no check, no branch — see `adapters-and-proxies.md`.

---

## Configuring the session instead

Everything you would have wanted a reachability check for is a `URLSessionConfiguration` setting.
**Set them before creating the session — later changes to the configuration have no effect on an
existing session.**

| Setting | Default | What it does |
|---|---|---|
| `waitsForConnectivity` | `false` | Instead of failing immediately, the task waits for connectivity to become available. **Ignored by background sessions** — they always wait |
| `timeoutIntervalForRequest` | **60 s** | Idle timeout, not a total cap — the timer resets whenever data arrives |
| `timeoutIntervalForResource` | **7 days** | How long the whole transfer (including any waiting) is allowed to take — set something sane (e.g. 600) if you enable waiting |
| `allowsCellularAccess` | `true` | Also settable per `URLRequest`. Note a reachability flag alone never guarantees traffic won't go over cellular |
| `allowsExpensiveNetworkAccess` | `true` | Interfaces the system considers expensive (cellular, personal hotspot) |
| `allowsConstrainedNetworkAccess` | `true` | Respects the user's **Low Data Mode** |
| `multipathServiceType` | `.none` | Multipath TCP — requires an entitlement, and is unavailable on macOS/watchOS/tvOS |
| `.ephemeral` | — | Nothing written to disk — no cache, credentials or session data. Use it when you own your caching strategy, and in tests |

These defaults were verified against the iOS 26.5 SDK documentation and headers.

Two behaviours worth knowing:

- **`waitsForConnectivity` only covers *establishing* the connection.** If the network drops
  mid-request, the request fails; it is not retried.
- **Background sessions ignore it** — they always wait for connectivity.
- It respects `allowsCellularAccess`: with cellular disallowed, the session waits for Wi-Fi.

Tell the customer what is happening rather than freezing the UI:

```swift
func urlSession(_ session: URLSession, taskIsWaitingForConnectivity task: URLSessionTask) {
    // Show "waiting for connection…" with a cancel button, or offer an offline mode
}
```

When a task does fail for one of these reasons, `networkUnavailableReason` on the error tells you
*which* (`.expensive`, `.constrained`) so you can retry, downgrade the request, or explain.

---

## URLSession retains its delegate

> `URLSession` holds a **strong** reference to its delegate until your app exits or you explicitly
> invalidate the session. **If you don't invalidate the session, your app leaks memory until it
> exits.**

This is the most commonly missed lifecycle rule in `URLSession` usage:

```swift
session.invalidateAndCancel()      // drops in-flight tasks, releases the delegate
session.finishTasksAndInvalidate() // lets in-flight tasks finish, then releases
```

The session cannot be used after invalidation. If your `URLSessionHTTPClient` takes a delegate, its
`deinit` is the natural place to invalidate — otherwise the leak-tracking tests will not catch it,
because the leak is in the *session*, not your type.

> Note this is exactly why the case study's client takes a plain `URLSession` and no delegate: no
> delegate, no retention problem.

---

## Enumerate representable states

Infrastructure APIs hand you loosely-typed tuples. Before writing a single test, **write the table**
of every combination and mark which are valid.

For `URLSession`'s `(Data?, URLResponse?, Error?)`:

| Data? | URLResponse? | Error? | Valid? |
|---|---|---|---|
| nil | nil | nil | ✗ |
| nil | nil | value | ✓ (request failed) |
| value | value | nil | ✓ (success) |
| …and five more | | | mostly ✗ |

Two lessons the table makes obvious:

- **Adding one optional doubles the table.** Two optionals give four states; three give eight, of
  which only two are valid.
- **Subtyping inflates it further.** Because `HTTPURLResponse` is a subclass of `URLResponse`, every
  "there is a response" row splits into "…and it's an HTTP response" and "…and it isn't" — more
  invalid states to defend against.

The fix is the same at every boundary: collapse to a type where the invalid states cannot be
written (`Result`, or an enum with exactly the valid cases). Then a single private
`UnexpectedValuesRepresentation: Error` covers the should-never-happen branch — the app must not
crash, and the customer must be able to retry.

> Migrating to the async `URLSession.data(from:)` API **shrinks this table structurally** — some
> invalid combinations become unrepresentable. Delete the tests for states that can no longer exist
> rather than contorting a stub to produce them.

---

## Persistence details

**Storing binary data in CoreData.** Enable `allowsExternalBinaryDataStorage` ("Allows External
Storage") and let CoreData decide between inline SQLite storage and an external file. The guidance
the course cites: keep values under ~100 KB comfortable in memory, and expect ~1 MB as the threshold
where external storage pays off. Large binary data in model objects makes them expensive to keep in
memory.

**Loading the model from the right bundle.** The `.xcdatamodeld` lives in the *production* bundle,
which is not the test bundle. Inject the bundle for tests and default to the main bundle in
production. And load the `NSManagedObjectModel` **once**, cached statically — loading it twice
registers duplicate `NSEntityDescription`s claiming the same `NSManagedObject` subclasses.

**`/dev/null` as the test store URL.** Points CoreData at the null device: no SQLite artifacts on
disk, all work in memory, faster (less I/O), and no cross-test cleanup because there is nothing to
clean. Prefer it to `NSInMemoryStoreType`, which needs
`shouldAddStoreAsynchronously = false` and `returnsObjectsAsFaults = false` to behave, and whose
named URLs are *shared* between stores in the same process.

**Keep framework conformances out of shared models.** A `Codable`/`@Model`/`Decodable` conformance
on a framework-agnostic model is the leak — mirror it with a private type inside the store. Full
argument in `architecture-layers.md`.

---

## Optimize in the infrastructure, not the service layer

> If you have a performance bottleneck, fix it in the infrastructure. If you have a good design,
> that is *easier* — easier to trace, easier to debug, easier to fix.

The use case stays "delete the old cache, insert the new one". You do not add caching cleverness to
business logic, because performance code in business logic is what becomes unmaintainable.

The worked example: re-saving a paginated feed re-downloaded every image, because deleting the cache
dropped the stored image data. The fix lives entirely inside the CoreData store —

- override `prepareForDeletion` on the managed image to stash its data in
  `managedObjectContext.userInfo`, keyed by URL,
- reuse it when inserting the new page,
- clear `userInfo` afterwards so RAM isn't held,
- and check that dictionary before querying the database (a constant-time lookup beats a query).

Not one line of the use case changed.

**Where duplicate-request prevention belongs.** An `isLoading` guard belongs in the shared
presentation adapter that *every* resource load passes through — not in one cell controller. Fix a
cross-cutting problem at the altitude where it is crossed. And note that cancellation never invokes
the completion, so the flag must also reset on cancel.

> **Measure first, and measure the right thing.** Use `CACurrentMediaTime()` for elapsed time, not
> `Date()` — wall-clock time gets synchronized and produces inconsistent measurements.

---

## Logging and graceful degradation

### Silent errors are the anti-pattern

```swift
do { try store.insert(feed) } catch { print(error) }   // ✗
```

Swallowing an error and continuing — often alongside optional dependencies and `guard let` bail-outs
— hides that the app is now running in a faulty state. Code that quietly ignores messages is *less*
safe, not more.

> **Safe code is not crash-free code.** Safe code does not let the system reach an invalid state; if
> it does reach one, crashing preserves integrity. A "99.9% crash-free" metric can be met by
> swallowing everything — and by apps whose users have to delete and reinstall.

### The three-part response

- **`assertionFailure`** — crashes in debug, no-op in release. Fast feedback during development
  without shipping the crash. Strictly better than `print`.
- **`Logger` (Apple's `os` framework)** — records the fault in release so an escape to production is
  traceable. `Logger(subsystem: "com.example.App", category: "main")`, category naming the module.
  Levels ascend Debug → Info → Notice/Default → Error → Fault, with increasing persistence; use
  `.fault` for "we should never get here". Library-agnostic — the same call sites work with
  Crashlytics or Datadog.
- **A fallback** — so the failure is not fatal. A store that fails to initialize degrades to
  in-memory rather than taking the app down.

### Null Object

Neutral behaviour instead of optionality: a `NullStore` whose deletes and inserts succeed and whose
retrieval always finds nothing. The neutral behaviour is a *design decision*, not a template. It
lives in the Composition Root and usually needs no tests, because it has no behaviour.

> **Prefer graceful degradation when the user would notice.** This codebase replaced its `NullStore`
> with `InMemoryFeedStore`: a null store silently drops every write and the customer loses caching
> entirely, whereas an in-memory store keeps the session working. Reach for a Null Object when doing
> nothing is genuinely correct.

The Null Object is the production sibling of the **dummy** test double, and a **fake** (like an
in-memory store) can be promoted out of tests into production — at which point it earns its own tests.

### Log from the Composition Root

Only the root knows the concrete types, so only it can decide what is worth observing. This is how
you avoid injecting a logger into every module or reaching for a global one. Logging is a
cross-cutting concern — apply it with a **decorator** (`adapters-and-proxies.md`), decorating the
specific instances you care about rather than everything.

The costs are real and worth stating:

- more code obscuring application code,
- logging is not free (disk writes, network sync),
- **the more you log, the less you can find**,
- and if it is worth saving to a log, ask whether it is worth showing in the UI.

> **Temporary diagnostic logs are not committed.** Once a test proves the behaviour, the log is
> noise. Keep permanent logs only for what you genuinely cannot test.

---

## Auth as a cross-cutting concern — the decorator you'll actually write

Authentication is the interception pattern's most common production use: a decorator that signs
every request and handles token expiry, composed once in the Composition Root
(`adapters-and-proxies.md` covers the pattern itself). Signing needs headers, so it presumes a
request-shaped client boundary (`perform(_: URLRequest)`) rather than the URL-only `get(from:)`:

```swift
protocol TokenProvider: Sendable {
    func validToken() async throws -> String
    func refreshedToken() async throws -> String
}

final class AuthenticatedHTTPClientDecorator: HTTPClient {
    private let decoratee: HTTPClient
    private let tokenProvider: TokenProvider

    init(decoratee: HTTPClient, tokenProvider: TokenProvider) {
        self.decoratee = decoratee
        self.tokenProvider = tokenProvider
    }

    func perform(_ request: URLRequest) async throws -> (Data, HTTPURLResponse) {
        let (data, response) = try await decoratee.perform(
            signed(request, with: try await tokenProvider.validToken()))
        guard response.statusCode == 401 else { return (data, response) }

        let freshToken = try await tokenProvider.refreshedToken()   // refresh once
        return try await decoratee.perform(signed(request, with: freshToken))  // retry once
    }

    private func signed(_ request: URLRequest, with token: String) -> URLRequest {
        var signed = request
        signed.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        return signed
    }
}
```

Because the decorator implements the same `HTTPClient` boundary it wraps, services and use cases
keep **zero** auth knowledge — the Composition Root decides which clients are authenticated.
Single-flight token refresh belongs *inside* the `TokenProvider` (make it an actor), so ten
concurrent 401s trigger exactly one refresh and the other nine await its result. Never retry more
than once: a second 401 after a fresh token is a real auth failure, and looping on it hammers the
backend and hides the bug.

---

## Guardrails

- Do not check reachability before making a request — attempt it and handle the result
- Do not cache a connectivity determination
- Do not mutate a `URLSessionConfiguration` after creating the session — it has no effect
- Do not create a delegate-based `URLSession` without invalidating it
- Do not write an infrastructure test suite before enumerating the representable states
- Do not put framework conformances on framework-agnostic models
- Do not swallow errors with `print` — assert in debug, log the fault in release, and degrade gracefully
- Do not inject loggers into feature modules — decorate at the composition site
- Do not optimize in the service or business layer — fix it in the infrastructure

## Verification

- [ ] No `SCNetworkReachability`/`NWPathMonitor` call gates a request
- [ ] Session configuration is fully set before the session is constructed
- [ ] Any delegate-based session is invalidated in `deinit`
- [ ] Every infrastructure boundary returns a type with no unrepresentable-state combinations
- [ ] The CoreData model is loaded once, from an injected bundle, and tests point at `/dev/null`
- [ ] Failure paths assert in debug, log a fault in release, and have a defined fallback
- [ ] No feature module imports a logging framework
