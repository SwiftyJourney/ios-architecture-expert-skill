# Concurrency at Architectural Boundaries — Code Reference

Use this when:
- Concurrency patterns affect module boundaries, composition wiring, or test infrastructure
- You need to understand where `@MainActor`, `@Sendable`, or `Sendable` annotations go
- You need the `Task.immediate` pattern for synchronous-first execution
- You need `async let` for parallel loading in the Composition Root
- You need cancellation handling in adapters
- You are enabling `SWIFT_STRICT_CONCURRENCY = complete`

Skip this file if:
- You need general Swift Concurrency guidance (actors, task groups, async sequences). Use the `swift-language-expert` skill (`references/concurrency-and-errors.md`).
- You need to wire adapters or proxies. Use `adapters-and-proxies.md`.
- You need Composition Root orchestration details. Use `composition-root.md`.

Jump to:
- [Asynchrony is an infrastructure detail](#asynchrony-is-an-infrastructure-detail)
- [Scheduler Protocol](#scheduler-protocol--abstract-execution-context)
- [@Sendable Closures at Composition Boundaries](#sendable-closures-at-composition-boundaries)
- [Task.immediate Pattern](#taskimmediate-pattern)
- [async let for Parallel Loading](#async-let-for-parallel-loading)
- [Cancellation Handling in Adapters](#cancellation-handling-in-adapters)
- [Swift 6 Strict Concurrency — Architectural Impact](#swift-6-strict-concurrency--architectural-impact)
- [Migration Playbook: Closures/Combine → async/await](#migration-playbook-closurescombine--asyncawait)
- [The limit: sync protocol requirements bypass actor isolation](#the-limit-sync-protocol-requirements-bypass-actor-isolation)
- [Swift 6 / UIKit Interop Gotchas](#swift-6--uikit-interop-gotchas)

---

## Asynchrony is an infrastructure detail

This section is the *why* behind `Scheduler`, and behind the fact that `FeedStore` is a
**synchronous `throws` protocol** in a codebase built on async/await. It looks backwards until you
see the argument.

### The leak

If a low-level component exposes an async API, every caller above it must handle asynchrony too —
so you end up making the whole call chain async because *one* implementation (CoreData) needs it.
That is a **leaky abstraction**: not all store implementations are async. An in-memory store is
not. Yet the async-ness of one implementation has infected the boundary, the use case, and
everything above.

Wrapping the boundary in `AnyPublisher` (or any other async-returning type) does not fix it — it
still leaks asynchrony *and* couples the domain abstraction to a concurrency framework.

### The rule

> **Prefer synchronous APIs in domain and infrastructure abstractions. Inject asynchrony at the
> composition site.**

Synchronous APIs are much simpler to implement, test and maintain, and you want high-level
components to be as simple as possible because that is where correctness matters most. Compare:

```swift
// Async boundary: nested completions, escaping closures, cancellation, dealloc tests
func save(_ feed: [FeedImage], completion: @escaping (Result<Void, Error>) -> Void) {
    store.deleteCachedFeed { [weak self] result in
        guard let self else { return }
        switch result { ... }          // and another level for insert
    }
}

// Sync boundary: the use case IS the recipe
public func save(_ feed: [FeedImage]) throws {
    try store.deleteCachedFeed()
    try store.insert(feed.toLocal(), timestamp: currentDate())
}
```

The second version deletes whole categories of test: "does not deliver result after the instance
was deallocated" and the cancellation tests **cannot happen anymore**. You get cancellation for
free from composition instead.

**This is not an anti-async position.** `HTTPClient` is legitimately async — the network genuinely
is. The point is to put asynchrony where it belongs rather than letting it propagate by default.

### Where the asynchrony goes instead

Two placements, and the difference matters:

| | Blocking | Non-blocking |
|---|---|---|
| Mechanism | `performAndWait` inside the low-level component | `perform { }` moved **up** to the composition site |
| Caller | Blocked — unacceptable on the main queue | Free |
| API shape | Sync | Sync |

The non-blocking form is what `Scheduler.schedule` implements: the component exposes a
synchronous API, and the **composer** decides which execution context wraps the call.

> **The database-transaction analogy.** A backend wraps a whole endpoint in a transaction at the
> composition level; the use case has no idea it is running in a transaction scope. Making the use
> case start, commit and roll back the transaction itself would push a low-level storage detail
> into business logic — and would break the day you use a storage mechanism without transactions.
> Wrapping store calls in a context queue is the same move.

### The payoff

> **Your components do not need to be thread-safe** if you control threading as a cross-cutting
> concern in the Composition Root.

That is the whole trade. One rule follows: **every operation on a non-thread-safe component must
go through the same scheduler.** A serial queue owned by the composer, or CoreData's own context
queue, is what makes that true. (CoreData *is* thread-safe as long as every access goes through
`perform`/`performAndWait`, which is why its scheduler may be concurrent.)

The same reasoning tells you what an infrastructure boundary should *not* decide: which thread or
executor its results arrive on. Document the contract on the protocol ("may complete on any
thread") and leave the hop to the client or the composer (003/013: "the clients should decide
because we don't know their needs").

**CQS pays a concurrency dividend.** Inside a stateful implementation, queries have no side effects
and can run concurrently; commands must be serialised. In the GCD era this was a reader-writer split
(concurrent queue, `retrieve` plain, `insert`/`delete` with a barrier — 003/013). Treat that as the
rationale and a legacy pattern, not the code to write today: with the sync store + `Scheduler`
design, the serial context or actor serialises reads too, and that is usually fine.

You can have as many schedulers as you want — one per domain module, or one for everything. It is
a composition decision, and it is also what lets acceptance tests run synchronously: in the case
study they inject the real store configured with `contextQueue: .main`, whose `schedule`
short-circuits without a hop — there is no separate scheduler object to inject.

**Know where a threading test belongs.** "Dispatches to the main thread" is a *composition*
concern, so it is tested in the UI integration/composition tests — not in the CoreData
implementation, which has no idea a UI exists.

---

## Scheduler Protocol — Abstract Execution Context

The Scheduler protocol abstracts whether store operations run on CoreData's background context or InMemory's main actor. Full code: `composition-root.md` (its canonical home — the protocol is a wiring concern, so it lives in the Composition Root, not the feature module).

Concurrency rationale:
- The CoreData conformance short-circuits **synchronously** when `contextQueue == .main` — no async hop, so acceptance tests injecting the real store with `contextQueue: .main` run deterministically. Background contexts dispatch through `NSManagedObjectContext.perform`.
- The InMemory conformance is always synchronous — the store is `@MainActor` with in-memory state.
- Store side effects that follow a network result are wrapped in `store.schedule { }` because, after `await httpClient.get`, the `@MainActor` `FeedService` is back on the main actor — `schedule` is what moves the cache work onto the store's (background) context instead of doing it on main. (In the closure/Combine era the rationale was different: get off the HTTP client's callback queue so cache writes didn't delay delivery of other in-flight responses.)

---

## `@Sendable` Closures at Composition Boundaries

The `@Sendable` annotation is required at composition boundaries where closures cross actor isolation domains. The most important example is the `loadMore` closure in pagination (full `makePage` code: `composition-root.md`):

```swift
{ @MainActor @Sendable in try await self.loadMoreRemoteFeed(last: last) }
```

**Why both annotations**:
- `@MainActor` — the closure must run on the main actor (composition code)
- `@Sendable` — the closure crosses actor boundaries (stored in `Paginated`, which is `Sendable`)

**Where these annotations go**: Only at the composition site, not in protocols. The `Paginated` struct defines `loadMore` as `@Sendable`:

```swift
public struct Paginated<Item: Sendable>: Sendable {
    public let items: [Item]
    public let loadMore: (@Sendable () async throws -> Self)?
}
```

The `@MainActor` annotation is added at the call site because it is a composition decision, not a type constraint.

### Scheduler closure annotations

```swift
// The action closure is @Sendable because it crosses from the @MainActor
// FeedService into potentially a different context (CoreData background)
await store.schedule { [store] in
    let localFeedLoader = LocalFeedLoader(store: store, currentDate: Date.init)
    try? localFeedLoader.save(feed)
}
```

Capture `[store]` rather than reading `self.store` inside the closure. Capturing `self` itself is legal — a `@MainActor` class is implicitly `Sendable`, and `makePage` above does exactly that. What fails is touching main-actor-isolated *state* from the non-isolated `@Sendable` closure: `store` and `logger` are `@MainActor` `lazy var`s, so `self.store` inside the action is "main actor-isolated property 'store' can not be referenced from a Sendable closure". The capture list `[store, logger]` reads those values on the main actor, before the closure runs, and hands the closure plain `Sendable` copies.

---

## `Task.immediate` Pattern

`Task.immediate` runs the closure synchronously up to the first suspension point (first `await`). A plain `Task` enqueues the whole body for a later turn.

### Why it matters

In `LoadResourcePresentationAdapter.loadResource()`, `didStartLoading()` is called **before** the Task is created, so it is synchronous either way — `Task.immediate` has nothing to do with it. What `Task.immediate` buys is that `loader()` starts in the same turn. Concurrency-relevant lines only (full adapter: `adapters-and-proxies.md`):

```swift
presenter?.didStartLoading()                              // synchronous — before the Task
cancellable = Task.immediate { @MainActor [weak self] in
    // runs now, up to the first real suspension: loader() is invoked in this turn
    if let resource = try await self?.loader() {
        if Task.isCancelled { return }                    // after every await
        self?.presenter?.didFinishLoading(with: resource)
    }
}
```

Two consequences the case study's tests depend on:

1. The request is registered synchronously. Integration tests assert `loader.loadFeedCallCount == 1` right after `simulateAppearance()`, with no `await` (`FeedUIIntegrationTests`).
2. With non-suspending stubs (`HTTPClientStub`) and a store whose `schedule` short-circuits (`contextQueue: .main`), the whole first load completes before `launch` returns, so acceptance tests assert rendered cells immediately (`FeedAcceptanceTests`). The same applies to `validateCache` below and the "deletes expired cache on entering background" test.

```swift
// With a bare Task, didStartLoading() still runs now, but loader() runs on a later turn —
// tests asserting the request count or rendered output synchronously fail.
cancellable = Task { @MainActor [weak self] in
    if let resource = try await self?.loader() { ... }
}
```

**Availability**: `Task.immediate` is SE-0472 (Implemented, Swift 6.2), with runtime availability aligned to **iOS 26 / macOS 26** and **no back-deployment**. On earlier deployment targets, guard it and fall back to plain `Task`:

```swift
#if compiler(>=6.2)
if #available(iOS 26.0, macOS 26.0, *) {
    Task.immediate { await self.load() }
} else {
    Task { await self.load() }
}
#else
Task { await self.load() }
#endif
```

On the fallback path the loader is *enqueued* rather than started synchronously. `didStartLoading()` is unaffected (it runs before the Task); what breaks is synchronous loader invocation and same-turn completion, so tests must await observable state instead. Never raise a user's deployment target just to get `Task.immediate`.

### Fire-and-forget with `Task.immediate`

```swift
func validateCache() {
    Task.immediate { @MainActor in
        await store.schedule { [store, logger] in
            do {
                let localFeedLoader = LocalFeedLoader(store: store, currentDate: Date.init)
                try localFeedLoader.validateCache()
            } catch {
                logger.error("Failed to validate cache with error: \(error.localizedDescription)")
            }
        }
    }
}
```

Called from `sceneWillResignActive` — starts immediately, errors are logged but not propagated.

---

## `async let` for Parallel Loading

When loading more feed items, cached items and new remote items can be fetched concurrently:

```swift
private func loadMoreRemoteFeed(last: FeedImage?) async throws -> Paginated<FeedImage> {
    async let cachedItems = try await loadLocalFeed()
    async let newItems = try await loadRemoteFeed(after: last)

    let items = try await cachedItems + newItems

    await store.schedule { [store] in
        let localFeedLoader = LocalFeedLoader(store: store, currentDate: Date.init)
        try? localFeedLoader.save(items)
    }

    return try await makePage(items: items, last: newItems.last)
}
```

**Architecture significance**: This reduces user-facing latency at the composition level. The feature modules (`LocalFeedLoader`, `HTTPClient`) are unaware of parallelism — the Composition Root decides the execution strategy.

**When to use `async let`**: When the number of concurrent operations is known at compile time and both results are needed before continuing. For dynamic counts, use `withTaskGroup`.

---

## Cancellation Handling in Adapters

Cancellation is an architectural concern — it prevents stale results from reaching the presenter after the view is deallocated.

### Three cancellation mechanisms

**1. Explicit check after `await`**:
```swift
if let resource = try await self?.loader() {
    if Task.isCancelled { return }  // Don't deliver stale result
    self?.presenter?.didFinishLoading(with: resource)
}
```

**2. Automatic cancellation on `deinit`**:
```swift
deinit { cancellable?.cancel() }
```

When the adapter is deallocated (view dismissed), in-flight tasks are cancelled automatically.
This compiles clean under Swift 6 strict concurrency only because it touches the object's own
`Sendable` stored property (`Task` is `Sendable`); touching any *other* `@MainActor` state from
`deinit` requires `isolated deinit` (SE-0371, Swift 6.2 compiler; no OS floor for `@MainActor`
classes, macOS 15.4/iOS 18.4 floor for plain actors).

**3. Explicit user-initiated cancellation**:
```swift
func didCancelImageRequest() {
    cancellable?.cancel()
    cancellable = nil
    isLoading = false
}
```

Called by the cell controller's `cancelLoad()`, which runs both from `didEndDisplaying` (the cell scrolled off screen) and from `cancelPrefetchingForRowsAt` (the row left the prefetch window).

### Why cancellation matters architecturally

Without cancellation checks, a slow network response could deliver results to a deallocated presenter (crash) or update UI that no longer represents the current screen state (visual glitch). The adapter layer is the right place for cancellation — it sits between the async loader and the presenter.

---

## Swift 6 Strict Concurrency — Architectural Impact

Enable strict concurrency across all targets. The case study's per-target settings (`project.pbxproj`):

| Target | `SWIFT_VERSION` | `SWIFT_STRICT_CONCURRENCY` | `SWIFT_APPROACHABLE_CONCURRENCY` | `SWIFT_DEFAULT_ACTOR_ISOLATION` |
|---|---|---|---|---|
| `EssentialFeed` (core framework) + unit, cache-integration, API end-to-end tests | 6 | `complete` | `YES` | nonisolated (default) |
| `EssentialFeediOS` (UIKit) | 6 | `complete` | `YES` | **`MainActor`** |
| `EssentialFeediOSTests` | 6 | `complete` | `YES` | nonisolated (default) |
| `EssentialApp` (Composition Root) + tests | 6 | `complete` | `YES` | nonisolated (default) |

Only the UIKit module defaults to `MainActor` — which is why `CellController` has to opt its
`Equatable`/`Hashable` conformances out with `nonisolated` (see
[the limit](#the-limit-sync-protocol-requirements-bypass-actor-isolation)). The app target, where
most of the wiring lives, stays nonisolated and annotates `FeedService`, composers and adapters
with `@MainActor` one by one — the approach recommended below for existing apps.

### Where `@MainActor` goes

| Layer | `@MainActor`? | Why |
|---|---|---|
| Domain models (`FeedImage`) | No | Value types, explicitly `Sendable` (see below) |
| Boundary protocols (`FeedStore`, `HTTPClient`) | No | Implementation decides isolation |
| Cache policy (`FeedCachePolicy`) | No | Pure functions, no mutable state |
| View protocols (`ResourceView`, `ResourceLoadingView`) | **Yes** | UI updates must happen on main thread |
| Presenters (`LoadResourcePresenter`) | **Yes** | Drives UI state transitions |
| Adapters (`LoadResourcePresentationAdapter`) | **Yes** | Bridges async to UI |
| View controllers / SwiftUI views | **Yes** | UIKit/SwiftUI requirement |
| Composition Root (`FeedService`, Composers) | **Yes** | All wiring on main thread |
| `InMemoryFeedStore` | **Yes** | Mutable state accessed from composition |

**This is a deliberate Swift 6 trade-off, not the course's original stance.** The course rejected
main-thread dispatch in the presenter because "the presenter is a platform-agnostic component.
And dispatching in the main thread is an implementation detail because of UIKit" (005/010). The
skill keeps that rule where it matters — a presenter never *dispatches*; hops happen at the
composition site — but accepts `@MainActor` on view protocols and presenters because every Apple
UI framework runs on the main actor and the annotation is checked at compile time, unlike runtime
dispatch. The cost: a non-main renderer (CLI, server-side, background snapshotting) would have to
hop onto the main actor to use the presenter.

Annotate PUBLIC view protocols `@MainActor` explicitly even when the target builds with `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` — the build setting is per-target and invisible to framework clients; the annotation makes isolation part of the API contract.

### Where `Sendable` goes

| Type | `Sendable`? | Mechanism |
|---|---|---|
| `FeedImage`, `ImageComment`, `ImageCommentsViewModel` | Yes | **Explicit** `Sendable` on a public `struct` whose stored properties are `Sendable` |
| `LocalFeedImage` | No | Public struct declared only `Equatable` — never crosses an isolation boundary, so it doesn't need to be |
| `Paginated<Item: Sendable>` | Yes | Explicit conformance, `loadMore` is `@Sendable` |
| `CoreDataFeedStore` | Yes | Explicit `Sendable` conformance, `let` properties |
| `InMemoryFeedStore` | Yes (implicit) | Global-actor-isolated classes are implicitly `Sendable` — which is why it satisfies `FeedStore & … & Sendable` in `FeedService` |

Implicit `Sendable` inference applies only to **non-public** structs and enums. A `public`
non-frozen struct is never implicitly `Sendable`, however simple its properties — write `Sendable`
explicitly on public value types that cross a boundary, and leave it off the ones that don't.

**Make a type `Sendable` only if it is meant to be used concurrently.** "Not all classes must be
concurrency safe. Only types you want to use concurrently must be concurrency safe" (008/001).
Don't force `HTTPClient` or every boundary protocol to be `Sendable` pre-emptively; add it where
a value actually crosses isolation domains (as the `store` existential in `FeedService` does).

### Newer tools at the boundary

Two additions to the vocabulary matter architecturally.

**Region-based isolation and `sending` (SE-0414/SE-0430).** The compiler now tracks *regions* of
values, not just types: a freshly created non-`Sendable` value that nothing else references sits in
a disconnected region and may legally cross an isolation boundary. This is exactly why the
`Scheduler.schedule<T>` signature needs **no** `T: Sendable` constraint — the closure creates its
result inside the store's context and hands it back as a fresh value, and the signature compiles
with zero warnings under complete checking even when `T` is non-`Sendable`. What region analysis
still rejects is capturing an *existing* main-actor-connected non-`Sendable` value into the
`@Sendable` closure — that value's region is already shared. Cosmetic note: `schedule` is
`rethrows`, so non-throwing call sites should drop the `try`.

**`isolated deinit` (SE-0371, Swift 6.2 compiler).** A plain `deinit` may only touch the object's
own `Sendable` stored properties (which is why `deinit { cancellable?.cancel() }` is fine); an
`isolated deinit` runs on the type's actor and may touch other isolated state. For `@MainActor`
classes it has no OS availability floor (the runtime thunk back-deploys); for plain actors it is
gated at macOS 15.4 / iOS 18.4.

---

## Migration Playbook: Closures/Combine → async/await

### The three ordered steps

1. **Turn on Complete concurrency checking everywhere and fix every warning — changing no APIs.**
2. **Replace completion closures, dispatch queues and Combine with async/await** where needed.
3. **Flip the Swift 6 language mode last.**

Swift 6 goes last deliberately: it is strict enough to generate both compile-time *and* runtime
errors, so it is far easier to reach a clean Complete-checking build first and only then turn it on.

**Migrate one target at a time**, taking a production target *together with its test target* as the
unit. Enabling checking across every target at once "generates thousands of warnings and build
errors — it can be extremely overwhelming." Work outward from the core framework, then the UI
framework, then the app, then the secondary test targets (end-to-end, integration), building the
whole workspace scheme before each commit.

Guiding constraints worth stating up front, because they shape every later decision:

- The solution must stay **fully testable**, and the tests must stay **just as fast and reliable**.
- **Preserve existing behaviour.** For an existing codebase it is safer to keep behaviour identical
  while changing technology.
- **Don't leak asynchrony everywhere** — handle concurrency in the Composition Root; keep components
  synchronous where it makes sense.
- No blocking in an async context; no unsafe/unchecked escapes outside a temporary bridge.

### Three build settings that decide a lot

| Setting | Values | How to choose |
|---|---|---|
| `SWIFT_STRICT_CONCURRENCY` | minimal / targeted / **complete** | Go to `complete` per target — that is step 1 |
| `SWIFT_DEFAULT_ACTOR_ISOLATION` | `nonisolated` / `MainActor` | `MainActor` is a good default for UI-heavy modules; `nonisolated` for mixed/non-UI ones |
| `SWIFT_APPROACHABLE_CONCURRENCY` | `NO` / `YES` | Xcode 26's recommended setting; among other things it enables `InferIsolatedConformances` (SE-0470). Know it is on — it changes which isolation errors you see (see [the limit](#the-limit-sync-protocol-requirements-bypass-actor-isolation)) |

New Xcode projects default to MainActor isolation. **For an existing app being migrated, do not flip
the app target wholesale** — "it will be a big change in behavior for our existing app, and things
can break unexpectedly." Annotate types one by one instead. The case study does exactly this: only
the UIKit module (`EssentialFeediOS`) uses `MainActor` by default; the core framework and the app
target stay nonisolated with explicit annotations (table in
[Swift 6 Strict Concurrency](#swift-6-strict-concurrency--architectural-impact)).

### Prove equivalence before deleting the old API

Before deleting a legacy closure/Combine method, re-implement it on top of the new async
implementation and run the **existing** tests against it. Green means the new code preserves
behaviour — "if the test pass with the new method, it means we kept the same behavior" (008/005).
Only then migrate callers and delete the old method. This is the opposite direction from the
deprecation bridge below (which wraps old in new to unblock callers); use both.

Migrate the **smallest feature first**, and harvest reusable building blocks along the way — a
generic async spy, an async `Scheduler` — so the bigger features reuse them: "deal with the
low-hanging fruit first" (008/004).

### Deprecation bridge for a protocol boundary (5 steps)

1. Add the async requirement PLUS a protocol-extension default implementation bridging to the old closure API via `withCheckedThrowingContinuation` wrapped in `withTaskCancellationHandler` (cancel the old task handle on Task cancellation). Mark the old requirement `@available(*, deprecated)`.
2. Migrate tests to the async API.
3. Migrate callers.
4. Implement the native async method in the infra type (`URLSession.data(from:)`, `context.perform`) — the default implementation is now dead code.
5. Delete the deprecated requirement and the bridge.

The deprecation warnings *are* the roadmap: they show you precisely which call sites remain, and
"only a warning left in the temporary helper" is how you know every client has been migrated.

`nonisolated(unsafe)` may appear ONLY inside this temporary bridge — it must be gone by step 5.

**Naming during coexistence**: prefix the NEW type (`AsyncLoadResourcePresentationAdapter`); give it the canonical name when the old one is deleted. Never rename the old one.

### Preserving timing, not just logic

Swift Concurrency can preserve *logical* behaviour while changing *timing* behaviour, and tests will
catch the difference. Two specifics:

- **The Combine pipeline started work synchronously at subscription; a plain `Task` enqueues it.**
  The course wrapped loaders in `Deferred { Future { … } }` precisely because an eager `Future`
  (firing on creation) was a problem — "we want to only fire a request when someone subscribes"
  (006/011). With `sink` called synchronously in the adapter, the request still started in the same
  turn. `Task.immediate` restores that synchronous start (the request is registered before
  `loadResource()` returns); plain `Task` defers it. The course's `ImmediateWhenOnMainQueueScheduler`
  was the same timing concern: an unconditional `receive(on: DispatchQueue.main)` always hops
  asynchronously, even when already on main.
- **A `Task` is not an `AnyCancellable`.** Cancellation on `deinit` was automatic with Combine; with
  tasks you must cancel manually, and you must check `Task.isCancelled` yourself after each `await`.

### Delete what the compiler now proves

A test that "completes on a background queue to ensure our production code is thread safe" is
obsolete once Complete checking is on. **Do not write tests for invariants the compiler enforces at
compile time** — delete them. Likewise, migrating to `URLSession.data(from:)` makes some invalid
`(data, response, error)` combinations structurally impossible; delete those assertions instead of
contorting a stub to produce a state that can no longer exist.

---

## The limit: sync protocol requirements bypass actor isolation

**What you see depends on one build setting, so know which world you are in.**

A `@MainActor` type conforming to a non-isolated **synchronous** protocol behaves differently with
and without Approachable Concurrency:

```swift
@MainActor public class InMemoryFeedStore { ... }
extension InMemoryFeedStore: FeedStore {               // FeedStore is NOT @MainActor
    func retrieve() throws -> CachedFeed? { ... }      // sync requirement, main-actor witness
}
```

- **Plain Swift 6** (no `InferIsolatedConformances`): this is a compile error —
  "conformance of 'InMemoryFeedStore' to protocol 'FeedStore' crosses into main actor-isolated code
  and can cause data races" `[#ConformanceIsolation]`. If a user reports that diagnostic, this is
  why.
- **Approachable Concurrency** (`SWIFT_APPROACHABLE_CONCURRENCY = YES`, Xcode 26's recommended
  setting, which turns on `InferIsolatedConformances`, SE-0470): the compiler silently infers an
  *isolated conformance* — `InMemoryFeedStore: @MainActor FeedStore`. It compiles. SE-0470 forbids
  using that conformance from outside the main actor, but once the value travels as
  `any FeedStore & Sendable` (exactly the type `FeedService` stores) and the sync requirement is
  called from a `@concurrent` function or `Task.detached`, you can get **no diagnostic and no
  runtime trap** — `retrieve()` runs off the main actor, touching main-actor state. A real data race.

Every case-study target enables Approachable Concurrency, which is the only reason
`extension InMemoryFeedStore: FeedStore` compiles there at all. So the hole is real in that
configuration — but it is a property of inferred isolated conformances, not of Swift 6 in general.

Your options, in the order you should consider them:

| Option | When it works | Cost |
|---|---|---|
| **Scheduler hop in the Composition Root** ✅ (what the case study does) | Always | `InMemoryFeedStore.schedule` is `@MainActor` and runs `action()` on main, so every sync store call happens on the right executor — concurrency stays confined to the root |
| Explicit isolated conformance (`extension InMemoryFeedStore: @MainActor FeedStore`) | You can keep the value out of `Sendable` / off-main contexts | Makes the isolation visible and checkable; breaks down as soon as the existential is erased to `& Sendable` and handed off-main |
| `nonisolated` witnesses (state behind a `Mutex`, or immutable) | The work is short, synchronous and genuinely thread-safe | You are back to making the component thread-safe yourself |
| Make the requirements `async` | You control the protocol | Restores the `await` boundary, but leaks asynchrony up the call chain |
| Isolate the protocol to an actor | *All* implementations genuinely share that isolation | Wrong for most infrastructure |

The same mechanism explains a UIKit-module detail: `EssentialFeediOS` is the only case-study
target built with `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`, so a plain
`extension CellController: Equatable, Hashable` would be inferred main-actor-isolated — but the
diffable data source compares and hashes items off the main queue. Hence
`extension CellController: nonisolated Equatable, nonisolated Hashable` with `nonisolated` `==` and
`hash(into:)` (`CellController.swift`).

**Do not isolate a store protocol.** CoreData has its own threading model, SQLite has another — "it's
not possible to define an actor isolation that will work well for all implementations, because
that's an implementation detail. And we don't want to leak implementation details in protocols."

**Do isolate view protocols.** `ResourceView`, `ResourceLoadingView` and `ResourceErrorView` are
`@MainActor` precisely because every implementation is UI code, and you want the compiler to stop
you sending messages to the UI off the main thread. Isolating a protocol cascades to its consumers —
the presenter using it becomes `@MainActor` too.

> It is a case-by-case decision. In another codebase a view protocol might *not* be a good candidate
> — for instance if you decorate it with types that shouldn't be main-actor isolated.

**The takeaway:** Swift Concurrency prevents data races *only if you annotate correctly*. It will
not save you from bad design or logical errors. Where the compiler cannot check something — such as
an inferred isolated conformance reached through a `Sendable` existential — **write a test for it.**
In the case study, exactly that test caught a race the compiler (with Approachable Concurrency on)
accepted silently.

---

## Swift 6 / UIKit Interop Gotchas

- `CALayer`/`CAGradientLayer` subclasses under `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` need `nonisolated override init()`, `nonisolated override init(layer:)`, `nonisolated required init?(coder:)` — otherwise the inits become MainActor-isolated and no longer override.
- KVO/notification callbacks are nonisolated; when they are known to arrive on main (e.g. `contentOffset` observation), wrap the body in `MainActor.assumeIsolated`.
- Migrating to async `URLSession.data(from:)` SHRINKS the error surface — some invalid `(data, response, error)` combinations become structurally impossible; delete those contract tests instead of stubbing impossible states.
- Under `@MainActor` isolation, "dispatches to main thread" tests become inexpressible — delete them; the type system now proves the property.
- `Thread.isMainThread` does not mean "on the main queue" (the main thread can run other queues); prefer `@MainActor` isolation over GCD-level checks.

---

## Guardrails

- Do not add `@MainActor` to domain types or store protocols — isolation is a composition decision
- Do not use `@unchecked Sendable` — redesign the type as a value type or use `@MainActor`
- Do not use `DispatchQueue`, semaphores, or locks (`Mutex` included) to protect state reached from async code — use actor isolation (`@MainActor` if the work is light, otherwise a dedicated actor). Reserve `Mutex` for short critical sections in synchronous code (e.g. legacy `URLProtocol` test stubs); never block an async context (008/004)
- Do not make a type `Sendable` unless it is meant to be used concurrently — and don't force every protocol (`HTTPClient`, stores) to be `Sendable` pre-emptively
- Do not read main-actor-isolated state (`self.store`, `self.logger`) inside a non-isolated `@Sendable` closure — capture the values instead (`[store, logger]`). Capturing a `@MainActor` `self` is legal; touching its isolated properties from the closure is not
- Do not make a domain or store boundary `async` because one implementation happens to be — keep it synchronous and inject the execution context at the composition site
- Do not make components thread-safe individually when the Composition Root can own threading as a cross-cutting concern — but if you take that route, route *every* call through the same scheduler
- Do not assume a `@MainActor` type stays isolated when reached through a non-isolated **synchronous** protocol requirement under Approachable Concurrency — the inferred isolated conformance can be called off-main through a `Sendable` existential without a warning; hop executors in the Composition Root. (In plain Swift 6 the same conformance is a `#ConformanceIsolation` error instead.)
- Do not isolate an infrastructure protocol to an actor — threading is an implementation detail and must not leak into the boundary
- Do not enable Complete checking on every target at once, and do not flip an existing app target to MainActor-by-default wholesale
- Do not keep tests for invariants the compiler now enforces, or for states the new API cannot represent
- Defer general concurrency questions (task groups, async sequences, actor reentrancy) to the `swift-language-expert` skill (`references/concurrency-and-errors.md`)

## Verification

- [ ] Build with `SWIFT_STRICT_CONCURRENCY = complete` on all targets — zero warnings
- [ ] All view protocols are `@MainActor`
- [ ] All presenters, adapters, and composition code are `@MainActor`
- [ ] No domain types or store protocols have `@MainActor`
- [ ] All `@Sendable` closures capture only `Sendable` values
- [ ] Run Thread Sanitizer in CI (xcodebuild flag `-enableThreadSanitizer YES`) — zero data races
- [ ] Run with `-com.apple.CoreData.ConcurrencyDebug 1` in test schemes — zero context-queue violations
- [ ] `Task.immediate` used where the loader must start in the same turn (iOS 26+ targets; earlier targets use plain `Task` with observable-state-based tests — never raise the deployment target for it)
- [ ] You know whether each target has `SWIFT_APPROACHABLE_CONCURRENCY` and `SWIFT_DEFAULT_ACTOR_ISOLATION` set — they decide which isolation diagnostics you get
- [ ] `Task.isCancelled` checked after every `await` in adapter code
