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

You can have as many schedulers as you want — one per domain module, or one for everything. It is
a composition decision, and it is also what lets acceptance tests inject an immediate,
main-queue scheduler and run synchronously.

**Know where a threading test belongs.** "Dispatches to the main thread" is a *composition*
concern, so it is tested in the UI integration/composition tests — not in the CoreData
implementation, which has no idea a UI exists.

---

## Scheduler Protocol — Abstract Execution Context

The Scheduler protocol abstracts whether store operations run on CoreData's background context or InMemory's main actor. Full code: `composition-root.md` (its canonical home — the protocol is a wiring concern, so it lives in the Composition Root, not the feature module).

Concurrency rationale:
- The CoreData conformance short-circuits **synchronously** when `contextQueue == .main` — no async hop, so acceptance tests injecting the real store with `contextQueue: .main` run deterministically. Background contexts dispatch through `NSManagedObjectContext.perform`.
- The InMemory conformance is always synchronous — the store is `@MainActor` with in-memory state.
- Store side effects that follow a network result are wrapped in `store.schedule { }` not only for thread safety but to get OFF the HTTP client's callback context — doing cache writes where the response arrives delays delivery of other in-flight responses.

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

Capture `[store]` — not `[self]`. The `FeedService` is `@MainActor` and cannot be captured in a `@Sendable` closure. Capturing just the `store` (which is `Sendable`) satisfies the compiler.

---

## `Task.immediate` Pattern

`Task.immediate` runs the closure synchronously up to the first suspension point (first `await`). This is critical for correct UI state sequencing.

### Why it matters

In `LoadResourcePresentationAdapter.loadResource()`, the presenter must receive `didStartLoading()` **synchronously** before any async work begins. This ensures the loading indicator appears immediately. Concurrency-relevant lines only (full adapter: `adapters-and-proxies.md`):

```swift
presenter?.didStartLoading()                              // synchronous — before the Task
cancellable = Task.immediate { @MainActor [weak self] in
    // ...
    if Task.isCancelled { return }                        // after every await
    self?.presenter?.didFinishLoading(with: resource)
}
```

```swift
// Incorrect: bare Task — didStartLoading() would run on the NEXT run loop iteration
cancellable = Task { @MainActor [weak self] in  // BAD
    self?.presenter?.didStartLoading()  // deferred — loading indicator appears late
    // ...
}
```

**Availability**: `Task.immediate` requires a Swift 6.2-era toolchain and recent deployment targets (the case study's deployment target is iOS 26). On older targets use plain `Task` and preserve the synchronous-`didStartLoading` guarantee another way (e.g. call `didStartLoading()` before creating the Task, as above).

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

**3. Explicit user-initiated cancellation**:
```swift
func didCancelImageRequest() {
    cancellable?.cancel()
    cancellable = nil
    isLoading = false
}
```

Called by `UITableViewDataSourcePrefetching` when a cell scrolls out of the prefetch window.

### Why cancellation matters architecturally

Without cancellation checks, a slow network response could deliver results to a deallocated presenter (crash) or update UI that no longer represents the current screen state (visual glitch). The adapter layer is the right place for cancellation — it sits between the async loader and the presenter.

---

## Swift 6 Strict Concurrency — Architectural Impact

Enable strict concurrency across all targets:

| Target | Setting |
|---|---|
| Feature framework | `SWIFT_STRICT_CONCURRENCY = complete` |
| iOS UI framework | `SWIFT_STRICT_CONCURRENCY = complete` |
| App target | `SWIFT_STRICT_CONCURRENCY = complete` |

### Where `@MainActor` goes

| Layer | `@MainActor`? | Why |
|---|---|---|
| Domain models (`FeedImage`) | No | Value types, automatically `Sendable` |
| Use-case protocols (`FeedCache`, `FeedStore`) | No | Implementation decides isolation |
| Cache policy (`FeedCachePolicy`) | No | Pure functions, no mutable state |
| View protocols (`ResourceView`, `ResourceLoadingView`) | **Yes** | UI updates must happen on main thread |
| Presenters (`LoadResourcePresenter`) | **Yes** | Drives UI state transitions |
| Adapters (`LoadResourcePresentationAdapter`) | **Yes** | Bridges async to UI |
| View controllers / SwiftUI views | **Yes** | UIKit/SwiftUI requirement |
| Composition Root (`FeedService`, Composers) | **Yes** | All wiring on main thread |
| `InMemoryFeedStore` | **Yes** | Mutable state accessed from composition |

Annotate PUBLIC view protocols `@MainActor` explicitly even when the target builds with `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` — the build setting is per-target and invisible to framework clients; the annotation makes isolation part of the API contract.

### Where `Sendable` goes

| Type | `Sendable`? | Mechanism |
|---|---|---|
| `FeedImage`, `ImageComment` | Yes | `struct` + all stored properties are `Sendable` |
| `LocalFeedImage` | Implicit | `struct` with only `Sendable` properties |
| `Paginated<Item: Sendable>` | Yes | Explicit conformance, `loadMore` is `@Sendable` |
| `CoreDataFeedStore` | Yes | Explicit `Sendable` conformance, `let` properties |
| `InMemoryFeedStore` | No | `@MainActor` provides isolation instead |
| `ResourceLoadingViewModel` | Implicit | `struct` with `Bool` property |

---

## Migration Playbook: Closures/Combine → async/await

Order matters — from the case study's migration history:

1. **Strictness first, APIs later.** Enable `SWIFT_STRICT_CONCURRENCY = complete` per target, inside-out (core framework → UI framework → app), and fix ALL warnings BEFORE changing any API. This forces `Sendable`/`@MainActor` onto existing code, so later API changes are purely structural.
2. **Leaf infrastructure protocols first.** Migrate `HTTPClient`, then the store — both have native async APIs to land on.
3. **Composition per feature, in two beats.** Beat 1: add the async functions in the Composition Root but keep the old adapter alive behind a bridge (historically `Deferred { Future { Task { … } } }`) — tests stay green. Beat 2: flip the adapter signature to `() async throws -> T` and delete the bridge.
4. **Flip the language mode last.** Delete the helpers, set `SWIFT_VERSION = 6.0`, and THEN annotate public view protocols `@MainActor` explicitly.

### Deprecation bridge for a protocol boundary (5 steps)

1. Add the async requirement PLUS a protocol-extension default implementation bridging to the old closure API via `withCheckedThrowingContinuation` wrapped in `withTaskCancellationHandler` (cancel the old task handle on Task cancellation). Mark the old requirement `@available(*, deprecated)`.
2. Migrate tests to the async API.
3. Migrate callers.
4. Implement the native async method in the infra type (`URLSession.data(from:)`, `context.perform`) — the default implementation is now dead code.
5. Delete the deprecated requirement and the bridge.

`nonisolated(unsafe)` may appear ONLY inside this temporary bridge — it must be gone by step 5.

**Naming during coexistence**: prefix the NEW type (`AsyncLoadResourcePresentationAdapter`); give it the canonical name when the old one is deleted. Never rename the old one.

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
- Do not use `DispatchQueue`, semaphores, or locks in async contexts — use actors or `Mutex`
- Do not capture `self` in `@Sendable` closures when `self` is `@MainActor` — capture specific `Sendable` properties instead (e.g., `[store]`, `[httpClient]`)
- Do not make a domain or store boundary `async` because one implementation happens to be — keep it synchronous and inject the execution context at the composition site
- Do not make components thread-safe individually when the Composition Root can own threading as a cross-cutting concern — but if you take that route, route *every* call through the same scheduler
- Defer general concurrency questions (task groups, async sequences, actor reentrancy) to the `swift-language-expert` skill (`references/concurrency-and-errors.md`)

## Verification

- [ ] Build with `SWIFT_STRICT_CONCURRENCY = complete` on all targets — zero warnings
- [ ] All view protocols are `@MainActor`
- [ ] All presenters, adapters, and composition code are `@MainActor`
- [ ] No domain types or store protocols have `@MainActor`
- [ ] All `@Sendable` closures capture only `Sendable` values
- [ ] Run Thread Sanitizer in CI (xcodebuild flag `-enableThreadSanitizer YES`) — zero data races
- [ ] Run with `-com.apple.CoreData.ConcurrencyDebug 1` in test schemes — zero context-queue violations
- [ ] `Task.immediate` used where synchronous-first execution is needed
- [ ] `Task.isCancelled` checked after every `await` in adapter code
