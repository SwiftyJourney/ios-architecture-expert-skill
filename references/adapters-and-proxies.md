# Adapters & Proxies — Code Reference

Use this when:
- You need to wire a `LoadResourcePresentationAdapter` to a presenter
- You need to break retain cycles with `WeakRefVirtualProxy`
- You need to map domain models to cell controllers via `FeedViewAdapter`
- You need to compose recursive pagination with `Paginated<Item>`
- You need to understand the `CellController` type-erasure pattern

Skip this file if:
- You need to understand `FeedService` orchestration or `Scheduler`. Use `composition-root.md`.
- You need concurrency rationale for `Task.immediate` or `@Sendable`. Use `concurrency-at-boundaries.md`.

Jump to:
- [LoadResourcePresentationAdapter](#loadresourcepresentationadapter)
- [WeakRefVirtualProxy](#weakrefvirtualproxy)
- [FeedViewAdapter](#feedviewadapter)
- [Paginated<Item>](#paginateditem)
- [CellController](#cellcontroller--type-erased-cell-composition)
- [Full Wiring Chain](#full-wiring-chain)

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
- `deinit` cancels in-flight tasks when the adapter is deallocated
- `didCancelImageRequest()` provides explicit cancellation for prefetching

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

**SwiftUI note**: `WeakRefVirtualProxy` is **not needed** with `@Observable`. SwiftUI manages the view lifecycle — when a view disappears, its `@Observable` view model is released automatically. The proxy pattern is UIKit-specific.

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

- Do not use bare `Task { }` in adapters — use `Task.immediate` for synchronous-first execution
- Do not skip `Task.isCancelled` checks after `await` — stale results must not reach the presenter
- Do not create adapters outside the Composition Root — they are wiring code, not business logic
- Do not hold strong references from presenter to view — always use `WeakRefVirtualProxy` (UIKit)
- Do not reuse `CellController` instances across different `FeedViewAdapter` lifetimes — let the `currentFeed` dictionary manage reuse within one page load cycle

## Verification

- [ ] Every `LoadResourcePresentationAdapter` checks `Task.isCancelled` after `await`
- [ ] Every `WeakRefVirtualProxy` wraps a view that could outlive the presenter
- [ ] `FeedViewAdapter` creates a new adapter instance for `loadMore` with forwarded `currentFeed`
- [ ] `Paginated.loadMore` is `nil` for the last page (when `last` item is `nil`)
- [ ] `CellController.id` uses the domain model (not an arbitrary UUID) for correct diffing
- [ ] `deinit` in `LoadResourcePresentationAdapter` cancels in-flight tasks
