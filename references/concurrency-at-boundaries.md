# Concurrency at Architectural Boundaries — Code Reference

Use this when:
- Concurrency patterns affect module boundaries, composition wiring, or test infrastructure
- You need to understand where `@MainActor`, `@Sendable`, or `Sendable` annotations go
- You need the `Task.immediate` pattern for synchronous-first execution
- You need `async let` for parallel loading in the Composition Root
- You need cancellation handling in adapters
- You are enabling `SWIFT_STRICT_CONCURRENCY = complete`

Skip this file if:
- You need general Swift Concurrency guidance (actors, task groups, async sequences). Use the `swift-concurrency` skill instead.
- You need to wire adapters or proxies. Use `adapters-and-proxies.md`.
- You need Composition Root orchestration details. Use `composition-root.md`.

Jump to:
- [Scheduler Protocol](#scheduler-protocol--abstract-execution-context)
- [@Sendable Closures at Composition Boundaries](#sendable-closures-at-composition-boundaries)
- [Task.immediate Pattern](#taskimmediate-pattern)
- [async let for Parallel Loading](#async-let-for-parallel-loading)
- [Cancellation Handling in Adapters](#cancellation-handling-in-adapters)
- [Swift 6 Strict Concurrency — Architectural Impact](#swift-6-strict-concurrency--architectural-impact)

---

## Scheduler Protocol — Abstract Execution Context

The Scheduler protocol abstracts whether store operations run on CoreData's background context or InMemory's main actor. The Composition Root decides the execution context — not the feature module.

```swift
protocol Scheduler {
    @MainActor
    func schedule<T>(_ action: @escaping @Sendable () throws -> T) async rethrows -> T
}
```

**Why this abstraction exists**: `FeedService` calls `store.schedule { ... }` without knowing whether the store is CoreData or InMemory. This is architectural polymorphism — the same code path works for both production and test environments.

### CoreData conformance

```swift
extension CoreDataFeedStore: Scheduler {
    @MainActor
    func schedule<T>(_ action: @escaping @Sendable () throws -> T) async rethrows -> T {
        if contextQueue == .main {
            return try action()
        } else {
            return try await perform(action)
        }
    }
}
```

When the context is on the main queue, the action runs synchronously (no async hop). When on a background queue, it dispatches through `NSManagedObjectContext.perform`.

### InMemory conformance

```swift
extension InMemoryFeedStore: Scheduler {
    @MainActor
    func schedule<T>(_ action: @escaping @Sendable () throws -> T) async rethrows -> T {
        try action()
    }
}
```

Always synchronous — `InMemoryFeedStore` is `@MainActor` and all data lives in memory.

---

## `@Sendable` Closures at Composition Boundaries

The `@Sendable` annotation is required at composition boundaries where closures cross actor isolation domains. The most important example is the `loadMore` closure in pagination:

```swift
// In FeedService.makePage
private func makePage(items: [FeedImage], last: FeedImage?) -> Paginated<FeedImage> {
    Paginated(items: items, loadMore: last.map { last in
        { @MainActor @Sendable in try await self.loadMoreRemoteFeed(last: last) }
    })
}
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

In `LoadResourcePresentationAdapter.loadResource()`, the presenter must receive `didStartLoading()` **synchronously** before any async work begins. This ensures the loading indicator appears immediately.

```swift
// Correct: Task.immediate — didStartLoading() runs synchronously
func loadResource() {
    guard !isLoading else { return }
    presenter?.didStartLoading()
    isLoading = true

    cancellable = Task.immediate { @MainActor [weak self] in
        defer { self?.isLoading = false }
        do {
            if let resource = try await self?.loader() {
                if Task.isCancelled { return }
                self?.presenter?.didFinishLoading(with: resource)
            }
        } catch {
            if Task.isCancelled { return }
            self?.presenter?.didFinishLoading(with: error)
        }
    }
}
```

```swift
// Incorrect: bare Task — didStartLoading() would run on the NEXT run loop iteration
cancellable = Task { @MainActor [weak self] in  // BAD
    self?.presenter?.didStartLoading()  // deferred — loading indicator appears late
    // ...
}
```

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

## Guardrails

- Do not add `@MainActor` to domain types or store protocols — isolation is a composition decision
- Do not use `@unchecked Sendable` — redesign the type as a value type or use `@MainActor`
- Do not use `DispatchQueue`, semaphores, or locks in async contexts — use actors or `Mutex`
- Do not capture `self` in `@Sendable` closures when `self` is `@MainActor` — capture specific `Sendable` properties instead (e.g., `[store]`, `[httpClient]`)
- Defer general concurrency questions (task groups, async sequences, actor reentrancy) to the `swift-concurrency` skill

## Verification

- [ ] Build with `SWIFT_STRICT_CONCURRENCY = complete` on all targets — zero warnings
- [ ] All view protocols are `@MainActor`
- [ ] All presenters, adapters, and composition code are `@MainActor`
- [ ] No domain types or store protocols have `@MainActor`
- [ ] All `@Sendable` closures capture only `Sendable` values
- [ ] Run Thread Sanitizer (`-enableThreadSanitizer YES`) — zero data races
- [ ] `Task.immediate` used where synchronous-first execution is needed
- [ ] `Task.isCancelled` checked after every `await` in adapter code
