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
- `didStartLoading()` is called before the Task exists, so it is synchronous regardless; `Task.immediate` makes `loader()` start in the same turn, so the request is registered synchronously and, with non-suspending stubs, tests can assert without awaiting (iOS 26+, no back-deployment — on earlier targets use plain `Task`; see `concurrency-at-boundaries.md`)
- `[weak self]` prevents retain cycles between the adapter and the task
- `isLoading` guard prevents duplicate requests
- `Task.isCancelled` checks after every `await` prevent stale results from reaching the presenter
- `deinit` cancels in-flight tasks when the adapter is deallocated. This compiles clean under Swift 6 strict concurrency only because it touches the object's own `Sendable` stored property (`Task` is `Sendable`); touching any *other* `@MainActor` state from `deinit` requires `isolated deinit` (SE-0371, Swift 6.2 compiler; no OS floor for `@MainActor` classes, macOS 15.4/iOS 18.4 floor for plain actors)
- `didCancelImageRequest()` provides explicit cancellation — the cell controller calls it when the cell ends displaying and when prefetching is cancelled
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

**Which references to proxy:** follow each reference the presenter holds — if it leads back to the
presenter (view → adapter → presenter → view), it closes a cycle and gets a `WeakRefVirtualProxy`;
if the presenter is its only owner, it stays strong. In `FeedUIComposer` the `ListViewController`
is proxied (it owns the adapter that owns the presenter), but the `FeedViewAdapter` is passed
strongly — proxy it and it deallocates immediately, and nothing renders: "making the feed view weak
would break the current composition" (005/007).

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

**Depend on the abstraction, not on concrete `Remote…`/`Local…` types.** A composite that takes
concrete loaders turns its unit tests into integration tests: add an access-token dependency to the
remote loader and unrelated composite tests break (006/002). The price is that both parameters now
have the same type, so the compiler can't catch swapped arguments — and that applies equally to the
closure form above, where two `() async throws -> [FeedImage]` parameters can be passed in the wrong
order silently. Cover it with behaviour tests ("what do you do when you don't have compile time
guarantees? You write tests"); marker protocols would restore the check, at the cost of extra types
the course chose not to add.

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

The right move is a **role-specific abstraction** for the save — a narrow capability with just
`save`, owned by the feature module — and the interceptor depends on that. With async/await the
narrowest form is often a function, `([FeedImage]) throws -> Void`, or the interceptor is a plain
function in the Composition Root as above; a single-method protocol earns its place only when a
named contract matters (see `design-principles.md` on protocol vs closure).

> History, so you don't copy a leftover: the case study still declares `FeedCache` (and
> `FeedImageDataLoader`/`FeedImageDataCache`), but each has one conformer (`LocalFeedLoader`,
> `LocalFeedImageDataLoader`) and no consumer — `FeedService` calls the concrete loaders inside
> `store.schedule`. There is no "loading boundary" any more either: the `FeedLoader` protocol was
> deleted in `fa5568b` once Combine's universal abstractions made it redundant. They were earned
> when the hand-rolled `FeedLoaderCacheDecorator` consumed them; today they are vestigial.

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

| Pattern | Hand-rolled type (closure era) | Combine era | async/await equivalent |
|---|---|---|---|
| Fallback composite | `FeedLoaderWithFallbackComposite` | `.fallback(to:)` | `do { try await primary() } catch { try await fallback() }` |
| Side-effect decorator | `FeedLoaderCacheDecorator` | `.caching(to:)` | A wrapping `async` function, or `map` over the result |
| Main-queue dispatch decorator | `MainQueueDispatchDecorator` | `.receive(on:)` with an immediate-when-on-main scheduler | `@MainActor` isolation — the type system proves it |
| Threading scheduler | Hand-rolled queue plumbing | `.subscribe(on:)` | `Scheduler.schedule` + actor isolation (`composition-root.md`) |

The names in the second column are the case study's historical types (replaced by Combine
operators in `12d8b4a`/`8672234`, 2020; the Combine was later replaced by async/await). Your own
generic equivalents would be `LoaderWithFallbackComposite`, `CacheDecorator`, and so on.

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

### The pagination contract

- **Each page carries ALL items loaded so far**, not just the new ones. Whoever builds the pages
  (the Composition Root) merges and de-duplicates; the UI never appends or dedups — "it's up to the
  clients that create the pages" (007/005).
- **The UI only calls `loadMore()`.** It doesn't know whether the backend is keyset- or
  offset-based, so you can switch between them without touching UI or presentation.
- **The cache knows nothing about pagination** — "that's an API detail." Each accumulated page is
  saved through the existing `save(items)`.
- **The cursor is the last item of the new page**; an empty new page gives `last == nil`, so
  `loadMore == nil`.
- **Never compute page keys client-side** (e.g. from sequential IDs) — that couples the client to
  the backend's key scheme.
- The case study's `async let` re-read of the cache alongside the remote page is an optional,
  measured memory optimisation, not part of the contract — the default is to accumulate in memory.
  If you mention it, state its risk: an expired cache or a failed save makes the re-read return
  fewer items than are on screen, so load-more silently drops pages.

**Load-more is content, not a list feature.** It is just another `CellController` in its own
section, built by `FeedViewAdapter`. Never add `hasPagination` / `showsRefresh` /
`separatorStyle`-style flags to a generic `ListViewController` — "suddenly there'll be like a
hundred booleans" (007/005). The generic type gets only what every client needs (or a platform
capability that can be switched off); per-feature behaviour is injected as content, and "whoever
creates it is responsible for decorating it according to its needs" (007/003). That's OCP: the list
is closed for modification, open through what you put in it.

Pagination UX acceptance criteria:

- [ ] Load-more triggers only after the first page has loaded, only when `loadMore != nil`, and never while a load-more request is in flight
- [ ] The load-more cell shows a loading state while loading
- [ ] A failed load-more shows a tappable error with retry — the user can never get stuck (connections fail)
- [ ] A failed load-more keeps the items already on screen
- [ ] The footer label is named for a generic "message", not "error", so it can carry other text

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

extension CellController: nonisolated Equatable {
    public nonisolated static func == (lhs: CellController, rhs: CellController) -> Bool {
        AnyHashable(lhs.id) == AnyHashable(rhs.id)
    }
}

extension CellController: nonisolated Hashable {
    public nonisolated func hash(into hasher: inout Hasher) {
        hasher.combine(AnyHashable(id))
    }
}
```

**Why `nonisolated` on the conformances**: `EssentialFeediOS` builds with
`SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` (and Approachable Concurrency), so a plain
`extension CellController: Equatable, Hashable` would be inferred as a main-actor-isolated
conformance. The diffable data source diffs on its own queue, off the main actor, so comparing and
hashing must not require it. Equality and hashing only read the immutable, `Sendable` `id`, so
opting out is safe. Any UI module that defaults to `MainActor` needs the same treatment for
conformances used by framework code off-main (see `concurrency-at-boundaries.md`, "The limit").

`CellController` enables `ListViewController` to manage any cell type through a uniform interface. The `id` property (typically the domain model itself, since it is `Hashable`) enables `DiffableDataSource` diffing.

**Why a struct, not a protocol**: a custom protocol couples feature modules to the shared UI module; a UIKit protocol-composition typealias forces clients to implement prefetch/delegate methods they don't need. The struct wraps any `UITableViewDataSource` and discovers optional capabilities via `as?` — Interface Segregation via composition.

### Cell lifecycle contract

Battle-tested against multiple iOS-version regressions:

- `cellForRowAt` must reset **all** mutable cell state (shimmer, retry-button visibility), not just the image — prefetch can race display.
- iOS 15+ may reuse a cached cell **without** calling `cellForRow` — implement `willDisplay` to recapture the cell reference and reload.
- Cells can be recycled without ever displaying, so `didEndDisplaying` may never fire — `prepareForReuse` must invoke an `onReuse` closure that releases the controller's cell reference, and the controller must nil that closure when it releases the cell (else the previous controller's reuse fires against the new controller's cell).
- In `didEndDisplaying` and `cancelPrefetchingForRowsAt`, look the controller up with `dataSource.itemIdentifier(for: indexPath)` — never by indexing your own model array, because `didEndDisplaying` fires for OLD cells after a data-source update. The diffable data source owns the old/new mapping; the `[IndexPath: CellController]` registry the case study once kept in `cellForRowAt` was deleted when it adopted diffable data sources (`e3fc04d`). Don't reintroduce it.

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
    |  (1) presenter?.didStartLoading()  [synchronous — called before the Task is created]
    |  (2) loader()                      [async — Task.immediate starts it in the same turn]
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
- Proxy with `WeakRefVirtualProxy` (UIKit) only the references that close a cycle back to the presenter (view → adapter → presenter → view); keep components owned only by the presenter (e.g. `FeedViewAdapter`) strong — proxying them deallocates them and nothing renders
- Do not add per-feature flags (`hasPagination`, `showsRefresh`) to a generic list — inject the behaviour as content (e.g. a load-more `CellController` section)
- Do not merge, append or dedup pages in the UI — each `Paginated` page already carries everything loaded so far
- Do not encode memory-management policy in a component — it belongs to whoever composes it
- Do not put a side effect inside a composite, and do not widen a boundary protocol to host one — introduce a narrow role-specific abstraction and intercept at the composition site
- Do not write a pattern type when a language construct already expresses it — write it when it removes real duplication
- Do not reuse `CellController` instances across different `FeedViewAdapter` lifetimes — let the `currentFeed` dictionary manage reuse within one page load cycle
- Closures stored on cells (`onRetry`, `selection`) must capture controllers weakly — a strong closure keeps the whole adapter→presenter chain alive across cell reuse

## Verification

- [ ] Every `LoadResourcePresentationAdapter` checks `Task.isCancelled` after `await`
- [ ] Every `WeakRefVirtualProxy` wraps a reference that closes a cycle back to the presenter, and no presenter-only-owned component (view adapter) is proxied
- [ ] `CellController` conformances used by the diffable data source are `nonisolated` in a `MainActor`-default UI module
- [ ] Cancellation in `didEndDisplaying` / `cancelPrefetching` looks controllers up via `dataSource.itemIdentifier(for:)`, with no `[IndexPath: CellController]` registry
- [ ] `FeedViewAdapter` creates a new adapter instance for `loadMore` with forwarded `currentFeed`
- [ ] `Paginated.loadMore` is `nil` for the last page (when `last` item is `nil`)
- [ ] `CellController.id` uses the domain model for ITEM cells (correct diffing/reuse); auxiliary cells with no domain identity (load-more) use a fresh `UUID()` so each page render replaces them
- [ ] `deinit` in `LoadResourcePresentationAdapter` cancels in-flight tasks
