# Composition Patterns — Adapters, Proxies, Composites & Decorators

Use this when:
- You need to wire a `LoadResourcePresentationAdapter` to a presenter
- You need to break retain cycles with `WeakRefVirtualProxy`
- You need to add behaviour to a type without changing it (Decorator), combine several types
  behind one interface (Composite), or inject a side effect at the composition site (Interception)
- You need to map domain models to cell controllers via `FeedViewAdapter`
- You need to compose recursive pagination with `Paginated<Item>`
- You need to understand the `CellController` type-erasure pattern

Skip this file if:
- You need to understand `FeedService` orchestration or `Scheduler`. Use `composition-root.md`.
- You need concurrency rationale for `Task.immediate` or `@Sendable`. Use `concurrency-at-boundaries.md`.
- You need *why* a pattern is justified rather than how to write it. Use `design-principles.md`.

Jump to:
- [Pattern selection](#pattern-selection)
- [LoadResourcePresentationAdapter](#loadresourcepresentationadapter)
- [WeakRefVirtualProxy](#weakrefvirtualproxy)
- [Composite](#composite)
- [Decorator](#decorator)
- [Interception](#interception)
- [Null Object](#null-object)
- [Patterns you no longer hand-roll](#patterns-you-no-longer-hand-roll)
- [FeedViewAdapter](#feedviewadapter)
- [Paginated<Item>](#paginateditem)
- [CellController](#cellcontroller--type-erased-cell-composition)
- [Full Wiring Chain](#full-wiring-chain)

---

## Pattern selection

Every pattern here is a variation on one move: **wrap a type in another type to change its
behaviour**. What differs is the intent.

| Intent | Pattern | Shape |
|---|---|---|
| Connect two APIs that don't match | **Adapter** | Implements A, holds a B, translates |
| Add behaviour, keep the interface | **Decorator** | Implements A, holds an A (the *decoratee*) |
| Combine several implementations behind one interface | **Composite** | Implements A, holds several As |
| Change lifetime/ownership without the client knowing | **Proxy** | Implements A, holds an A weakly |
| Supply neutral behaviour instead of nothing | **Null Object** | Implements A, does nothing meaningfully |

> Rule of thumb: **decorator** when you want to add behaviour to a type without changing it;
> **composite** when you want to compose types that conform to a common interface.

All of them live in the **Composition Root**. They are wiring, not business logic.

---

## `LoadResourcePresentationAdapter`

Generic adapter bridging async use cases to the presenter's `didStartLoading` / `didFinishLoading` callbacks. Supports cancellation.

```swift
@MainActor
final class LoadResourcePresentationAdapter<Resource, View: ResourceView> {
    private let loader: () async throws -> Resource
    private var cancellable: Task<Void, Never>?
    private var isLoading = false

    var presenter: LoadResourcePresenter<Resource, View>?

    init(loader: @escaping () async throws -> Resource) {
        self.loader = loader
    }

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

    deinit { cancellable?.cancel() }
}

// Conform to cell controller delegate for image loading
extension LoadResourcePresentationAdapter: FeedImageCellControllerDelegate {
    func didRequestImage() { loadResource() }
    func didCancelImageRequest() {
        cancellable?.cancel()
        cancellable = nil
        isLoading = false
    }
}
```

**Key details**:
- `Task.immediate` ensures `didStartLoading()` runs synchronously before the async work begins
- `[weak self]` prevents retain cycles between the adapter and the task
- `isLoading` guard prevents duplicate requests
- `Task.isCancelled` checks after every `await` prevent stale results from reaching the presenter
- `deinit` cancels in-flight tasks when the adapter is deallocated. This compiles clean under Swift 6 strict concurrency only because it touches the object's own `Sendable` stored property (`Task` is `Sendable`); touching any *other* `@MainActor` state from `deinit` requires `isolated deinit` (SE-0371, Swift 6.2 compiler; no OS floor for `@MainActor` classes, macOS 15.4/iOS 18.4 floor for plain actors)
- `didCancelImageRequest()` provides explicit cancellation for prefetching
- `presenter` is deliberately a `var` (property injection) — the adapter→presenter→view→adapter cycle can only be closed after construction, in the Composition Root. Everywhere else prefer constructor injection.

---

## `WeakRefVirtualProxy`

Generic proxy using conditional conformance to break retain cycles between presenters and views.

```swift
final class WeakRefVirtualProxy<T: AnyObject> {
    private weak var object: T?

    init(_ object: T) {
        self.object = object
    }
}

extension WeakRefVirtualProxy: ResourceErrorView where T: ResourceErrorView {
    func display(_ viewModel: ResourceErrorViewModel) {
        object?.display(viewModel)
    }
}

extension WeakRefVirtualProxy: ResourceLoadingView where T: ResourceLoadingView {
    func display(_ viewModel: ResourceLoadingViewModel) {
        object?.display(viewModel)
    }
}

extension WeakRefVirtualProxy: ResourceView where T: ResourceView, T.ResourceViewModel == UIImage {
    func display(_ model: UIImage) {
        object?.display(model)
    }
}
```

**Why it exists**: removing `weak var` from the presenter removes the `AnyObject` constraint from view protocols and moves lifetime policy to the Composition Root — the presenter cannot leak, and cannot even know views are held weakly.

The full argument matters, because it generalizes: `weak var view` on the presenter would let a
*composition* detail dictate the presenter's *API*, and **different compositions need different
memory rules** — the same presenter needs its view held weakly in one wiring and strongly in
another (the view adapter the composer owns). A rule baked into the presenter cannot be right for
both. Memory management is a responsibility of the composer. See the full reasoning in
[presentation-patterns.md](presentation-patterns.md#memory-management-belongs-in-the-composer).

**SwiftUI note**: `WeakRefVirtualProxy` is **not needed** with `@Observable`. SwiftUI manages the view lifecycle — when a view disappears, its `@Observable` view model is released automatically. The proxy pattern is UIKit-specific.

---

## Composite

Compose several objects that share a common interface behind *the same* interface, so clients
treat the composition uniformly and never learn it is a composition.

```swift
// Fallback strategy: try primary, fall back on failure.
func loadWithFallback(
    primary: @escaping () async throws -> [FeedImage],
    fallback: @escaping () async throws -> [FeedImage]
) -> () async throws -> [FeedImage] {
    return {
        do { return try await primary() }
        catch { return try await fallback() }
    }
}
```

**Why a composite and not an `if/else` at the call site**:

- No counters, no state, no branching leaks into either loader — they stay unaware of each other.
- **The nesting *is* the algorithm.** Reading the composition in the Composition Root tells you the
  strategy; reading a branch buried in a loader does not.
- **Composites compose infinitely.** Because a composite implements the same interface, it can be
  the primary or the fallback of another composite: in-memory → disk → remote is three nested
  composites, no new type.
- **Retry needs no state**: nest the same loader as its own fallback N times.

Naming carries the flexibility. `remote`/`local` describe *this* use; `primary`/`fallback`
describe the pattern, and that rename is what makes the type genuinely reusable. Suffix the type
with `Composite` so readers know the semantics.

**Composition asymmetry is a real decision**, not an oversight: feed loading is remote-first
(freshness matters), image loading is local-first (image data per URL is immutable and expensive
to re-download, so a cache hit is always valid).

> In this codebase the fallback is expressed directly with `do/catch` inside `FeedService` — see
> [Patterns you no longer hand-roll](#patterns-you-no-longer-hand-roll). Reach for an explicit
> composite type when the strategy is reused across several compositions.

---

## Decorator

Add behaviour to an instance **while keeping the same interface**. The decorator holds a
*decoratee* of the same type and forwards to it.

```swift
final class ProfilingHTTPClientDecorator: HTTPClient {
    private let decoratee: HTTPClient
    private let logger: Logger

    init(decoratee: HTTPClient, logger: Logger) {
        self.decoratee = decoratee
        self.logger = logger
    }

    func get(from url: URL) async throws -> (Data, HTTPURLResponse) {
        let start = CACurrentMediaTime()          // not Date — see composition-root.md
        do {
            let result = try await decoratee.get(from: url)
            logger.trace("finished \(url) in \(CACurrentMediaTime() - start)s")
            return result
        } catch {
            logger.error("failed \(url): \(error.localizedDescription)")
            throw error
        }
    }
}
```

**This is the Open/Closed Principle in practice** — behaviour added without changing (or
recompiling, or redeploying) the decorated type. A change in a UI or logging requirement must not
require changes in a networking or database module.

**Why the composition layer is the right place** for cross-cutting concerns:

- Only the composer knows the concrete implementations, so only it can decide whether the
  behaviour is needed. A presenter dispatching to the main queue, or a use case logging, is
  leaking an implementation detail upward.
- Duplicated defensive code across UI components — every one of them re-checking a thread, or
  logging the same way — **is the symptom** that the logic belongs one level up.
- You can decorate **specific instances**, not the whole app: profile one client, log errors only
  in production, leave the rest untouched.

If you find yourself writing several decorators, make one generic over its decoratee and put each
conformance in a conditional extension:

```swift
final class LoggingDecorator<T> {
    let decoratee: T
    let logger: Logger
    init(decoratee: T, logger: Logger) { self.decoratee = decoratee; self.logger = logger }
}

extension LoggingDecorator: HTTPClient where T: HTTPClient { ... }
extension LoggingDecorator: FeedStore where T: FeedStore { ... }
```

---

## Interception

**Interception** is the technique the Decorator enables: intercept a message at the composition
site and inject extra behaviour or a side effect, so *neither* collaborator knows it happened.

The canonical case is cache-on-success. Loading is a **query**; saving is a **command** with side
effects. CQS says keep them apart — so keep them apart, and *compose* them:

```swift
func loadAndCacheRemoteFeed() async throws -> [FeedImage] {
    let feed = try await loadRemoteFeed()
    try? cache.save(feed)        // intercepted side effect — fire-and-forget by design
    return feed
}
```

Two alternatives that look reasonable and are not:

| Rejected | Why |
|---|---|
| Put the save **inside the composite** | The composite now depends on a concrete cache type — it stops being composable |
| Add `save` to the loader boundary | ISP violation (loaders are forced to implement saving) and arguably LSP — not every loader can save |

The right move is a **role-specific abstraction**: a narrow `FeedCache` protocol with just `save`,
owned by the feature module. The interceptor depends on that; the concrete `LocalFeedLoader`
conforms to both it and the loading boundary.

Rules that keep interception honest:

- **The interceptor coordinates; it does not implement.** It does not cache — it calls something
  that caches.
- **Only mutate on success.** A failed load must not write anything, and that deserves its own
  test.
- **Make "ignore the result" explicit.** A `saveIgnoringResult` helper (or a deliberate `try?` with
  a comment) documents that the discard is intentional, not a forgotten error path.

---

## Null Object

An object with neutral behaviour, supplied instead of nothing:

```swift
final class NullStore: FeedStore {
    func deleteCachedFeed() throws {}
    func insert(_ feed: [LocalFeedImage], timestamp: Date) throws {}
    func retrieve() throws -> CachedFeed? { nil }
}
```

It exists so callers never branch on optionality. **The neutral behaviour differs per use case** —
here "retrieval always finds nothing" — so it is a design decision, not a template. It lives in the
Composition Root and usually needs no tests, because it has no behaviour.

**When to prefer graceful degradation instead.** This codebase replaced its `NullStore` with
`InMemoryFeedStore`: if CoreData fails to initialize, a null store silently drops every write and
the customer loses caching entirely, whereas an in-memory store keeps the session working. Reach
for a Null Object when doing nothing is genuinely correct; reach for a real fallback when the user
would notice. See `composition-root.md`.

> The Null Object is the production sibling of the **dummy** test double — a collaborator supplied
> only to satisfy a signature.

---

## Patterns you no longer hand-roll

Several of these patterns are specializations of operations the language already provides. Keep
the **vocabulary** — it is how you communicate intent — but skip the boilerplate when a built-in
does the job:

| Pattern | Hand-rolled type | Built-in equivalent |
|---|---|---|
| Fallback composite | `LoaderWithFallbackComposite` | `do { try await primary() } catch { try await fallback() }` |
| Side-effect decorator | `CacheDecorator` | A wrapping `async` function, or `map` over the result |
| Main-queue dispatch decorator | `MainQueueDispatchDecorator` | `@MainActor` isolation — the type system proves it |
| Threading scheduler | Hand-rolled queue plumbing | `Scheduler.schedule` + actor isolation (`composition-root.md`) |

The judgement call is reuse: a `do/catch` inline in one composer is clearer than a type; the same
fallback needed in four compositions earns the type. Write the type when it removes duplication,
not because the pattern has a name.

> Under `@MainActor` isolation, "dispatches to the main thread" tests become inexpressible — delete
> them, the compiler now proves the property. See `concurrency-at-boundaries.md`.

---

## `FeedViewAdapter`

Maps domain models to `CellController` arrays. Reuses existing cell controllers for unchanged items. Composes recursive `loadMore` pagination.

```swift
@MainActor
final class FeedViewAdapter: ResourceView {
    private weak var controller: ListViewController?
    private let imageLoader: (URL) async throws -> Data
    private let selection: (FeedImage) -> Void
    private let currentFeed: [FeedImage: CellController]

    init(currentFeed: [FeedImage: CellController] = [:],
         controller: ListViewController,
         imageLoader: @escaping (URL) async throws -> Data,
         selection: @escaping (FeedImage) -> Void) {
        self.currentFeed = currentFeed
        self.controller = controller
        self.imageLoader = imageLoader
        self.selection = selection
    }

    func display(_ viewModel: Paginated<FeedImage>) {
        guard let controller = controller else { return }

        var currentFeed = self.currentFeed
        let feed: [CellController] = viewModel.items.map { model in
            if let controller = currentFeed[model] { return controller }

            let adapter = LoadResourcePresentationAdapter<Data, WeakRefVirtualProxy<FeedImageCellController>>(
                loader: { [imageLoader] in try await imageLoader(model.url) }
            )

            let view = FeedImageCellController(
                viewModel: FeedImagePresenter.map(model),
                delegate: adapter,
                selection: { [selection] in selection(model) })

            adapter.presenter = LoadResourcePresenter(
                resourceView: WeakRefVirtualProxy(view),
                loadingView: WeakRefVirtualProxy(view),
                errorView: WeakRefVirtualProxy(view),
                mapper: UIImage.tryMake)

            let controller = CellController(id: model, view)
            currentFeed[model] = controller
            return controller
        }

        // Recursive loadMore composition
        guard let loadMoreAsync = viewModel.loadMore else {
            controller.display(feed)
            return
        }

        let loadMoreAdapter = LoadResourcePresentationAdapter<Paginated<FeedImage>, FeedViewAdapter>(
            loader: loadMoreAsync)

        let loadMore = LoadMoreCellController(callback: loadMoreAdapter.loadResource)

        loadMoreAdapter.presenter = LoadResourcePresenter(
            resourceView: FeedViewAdapter(
                currentFeed: currentFeed,
                controller: controller,
                imageLoader: imageLoader,
                selection: selection),
            loadingView: WeakRefVirtualProxy(loadMore),
            errorView: WeakRefVirtualProxy(loadMore))

        controller.display(feed, [CellController(id: UUID(), loadMore)])
    }
}
```

**Key insight**: The `loadMore` adapter's `resourceView` is a **new `FeedViewAdapter`** that carries forward the `currentFeed` dictionary. This enables cell reuse across pagination loads while recursively composing the next `loadMore`.

After a load-more error, retry-on-scroll is implemented by KVO-observing `tableView.contentOffset` while the load-more cell is displayed (installed in `willDisplay`, torn down in `didEndDisplaying`), guarded by `tableView.isDragging` to avoid programmatic-scroll retry loops.

---

## `Paginated<Item>`

Generic recursive pagination model. `loadMore` is optional — `nil` means no more pages.

```swift
public struct Paginated<Item: Sendable>: Sendable {
    public let items: [Item]
    public let loadMore: (@Sendable () async throws -> Self)?

    public init(items: [Item], loadMore: (@Sendable () async throws -> Self)? = nil) {
        self.items = items
        self.loadMore = loadMore
    }
}
```

**Usage in Composition Root**:

```swift
private func makePage(items: [FeedImage], last: FeedImage?) -> Paginated<FeedImage> {
    Paginated(items: items, loadMore: last.map { last in
        { @MainActor @Sendable in try await self.loadMoreRemoteFeed(last: last) }
    })
}
```

The `last` item determines if there are more pages. When `last` is `nil`, `loadMore` is `nil` — pagination ends. The closure captures `self` (the `FeedService`) to enable recursive page loading.

---

## `CellController` — Type-Erased Cell Composition

Wraps any cell's data source, delegate, and prefetching behavior behind a single value:

```swift
public struct CellController {
    let id: any Hashable & Sendable
    let dataSource: UITableViewDataSource
    let delegate: UITableViewDelegate?
    let dataSourcePrefetching: UITableViewDataSourcePrefetching?

    public init(id: any Hashable & Sendable, _ dataSource: UITableViewDataSource) {
        self.id = id
        self.dataSource = dataSource
        self.delegate = dataSource as? UITableViewDelegate
        self.dataSourcePrefetching = dataSource as? UITableViewDataSourcePrefetching
    }
}

extension CellController: Equatable, Hashable {
    // hash/compare by id — enables DiffableDataSource
}
```

`CellController` enables `ListViewController` to manage any cell type through a uniform interface. The `id` property (typically the domain model itself, since it is `Hashable`) enables `DiffableDataSource` diffing.

**Why a struct, not a protocol**: a custom protocol couples feature modules to the shared UI module; a UIKit protocol-composition typealias forces clients to implement prefetch/delegate methods they don't need. The struct wraps any `UITableViewDataSource` and discovers optional capabilities via `as?` — Interface Segregation via composition.

### Cell lifecycle contract

Battle-tested against multiple iOS-version regressions:

- `cellForRowAt` must reset **all** mutable cell state (shimmer, retry-button visibility), not just the image — prefetch can race display.
- iOS 15+ may reuse a cached cell **without** calling `cellForRow` — implement `willDisplay` to recapture the cell reference and reload.
- Cells can be recycled without ever displaying, so `didEndDisplaying` may never fire — `prepareForReuse` must invoke an `onReuse` closure that releases the controller's cell reference, and the controller must nil that closure when it releases the cell (else the previous controller's reuse fires against the new controller's cell).
- Cancel requests via the loading-controllers registry (`[IndexPath: CellController]` populated in `cellForRowAt`), never by indexing the new model — `didEndDisplaying` fires for OLD cells after a data-source update (see SKILL.md gotcha).

---

## Full Wiring Chain

```
User pulls to refresh
    |
    v
ListViewController.onRefresh
    |
    v
LoadResourcePresentationAdapter.loadResource()
    |  (1) presenter?.didStartLoading()  [synchronous via Task.immediate]
    |  (2) loader()                      [async — calls FeedService]
    v
FeedService.loadRemoteFeedWithLocalFallback()
    |  - tries remote, falls back to local
    |  - returns Paginated<FeedImage>
    v
LoadResourcePresenter.didFinishLoading(with: resource)
    |  - calls mapper (identity for Paginated)
    v
FeedViewAdapter.display(Paginated<FeedImage>)
    |  - maps each FeedImage to CellController
    |  - composes LoadMoreCellController if loadMore != nil
    v
ListViewController.display(feed, [loadMore])
    |  - applies DiffableDataSource snapshot
    v
UITableView renders cells
```

---

## Guardrails

- Prefer `Task.immediate` over bare `Task { }` in adapters for synchronous-first execution — noting its availability floor (iOS 26+, no back-deployment); on earlier targets use plain `Task` and test via observable state (see `concurrency-at-boundaries.md`)
- Do not skip `Task.isCancelled` checks after `await` — stale results must not reach the presenter
- Do not create adapters, composites, decorators or proxies outside the Composition Root — they are wiring code, not business logic
- Do not hold strong references from presenter to view — always use `WeakRefVirtualProxy` (UIKit)
- Do not encode memory-management policy in a component — it belongs to whoever composes it
- Do not put a side effect inside a composite, and do not widen a boundary protocol to host one — introduce a narrow role-specific abstraction and intercept at the composition site
- Do not write a pattern type when a language construct already expresses it — write it when it removes real duplication
- Do not reuse `CellController` instances across different `FeedViewAdapter` lifetimes — let the `currentFeed` dictionary manage reuse within one page load cycle
- Closures stored on cells (`onRetry`, `selection`) must capture controllers weakly — a strong closure keeps the whole adapter→presenter chain alive across cell reuse

## Verification

- [ ] Every `LoadResourcePresentationAdapter` checks `Task.isCancelled` after `await`
- [ ] Every `WeakRefVirtualProxy` wraps a view that could outlive the presenter
- [ ] `FeedViewAdapter` creates a new adapter instance for `loadMore` with forwarded `currentFeed`
- [ ] `Paginated.loadMore` is `nil` for the last page (when `last` item is `nil`)
- [ ] `CellController.id` uses the domain model for ITEM cells (correct diffing/reuse); auxiliary cells with no domain identity (load-more) use a fresh `UUID()` so each page render replaces them
- [ ] `deinit` in `LoadResourcePresentationAdapter` cancels in-flight tasks
