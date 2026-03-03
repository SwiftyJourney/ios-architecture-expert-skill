# Composition & Adapters — Code Reference

## Composition Root (`FeedService`)

The Composition Root is the **only place** that knows about all modules. It creates concrete instances, wires dependencies, and defines fallback strategies.

```swift
@MainActor
final class FeedService {
    // Lazy init — defer expensive creation until first use
    private lazy var httpClient: HTTPClient = {
        URLSessionHTTPClient(session: URLSession(configuration: .ephemeral))
    }()

    private lazy var store: FeedStore & FeedImageDataStore & Scheduler & Sendable = {
        do {
            return try CoreDataFeedStore(
                storeURL: NSPersistentContainer
                    .defaultDirectoryURL()
                    .appendingPathComponent("feed-store.sqlite"))
        } catch {
            assertionFailure("Failed to instantiate CoreData store: \(error.localizedDescription)")
            logger.fault("Failed to instantiate CoreData store: \(error.localizedDescription)")
            return InMemoryFeedStore()  // Fallback — graceful degradation
        }
    }()

    private lazy var baseURL = URL(string: "https://ile-api.essentialdeveloper.com/essential-feed")!

    // Convenience init for testing — inject all dependencies
    convenience init(httpClient: HTTPClient,
                     store: FeedStore & FeedImageDataStore & Scheduler & Sendable) {
        self.init()
        self.httpClient = httpClient
        self.store = store
    }
}
```

**Key principles**:
- `lazy` properties defer creation and allow override in tests via `convenience init`
- Protocol composition (`FeedStore & FeedImageDataStore & Scheduler & Sendable`) constrains the store type at the wiring site
- Fallback pattern: CoreData → InMemory on failure
- `@MainActor` ensures all composition happens on the main thread

### Remote-with-local-fallback pattern

```swift
func loadRemoteFeedWithLocalFallback() async throws -> Paginated<FeedImage> {
    do {
        let feed = try await loadAndCacheRemoteFeed()
        return makeFirstPage(items: feed)
    } catch {
        let feed = try await loadLocalFeed()
        return makeFirstPage(items: feed)
    }
}

private func loadAndCacheRemoteFeed() async throws -> [FeedImage] {
    let feed = try await loadRemoteFeed()
    await store.schedule { [store] in
        let loader = LocalFeedLoader(store: store, currentDate: Date.init)
        try? loader.save(feed)
    }
    return feed
}
```

### Scheduler protocol — abstract execution context

```swift
protocol Scheduler {
    @MainActor
    func schedule<T>(_ action: @escaping @Sendable () throws -> T) async rethrows -> T
}

// CoreData: dispatch to context queue
extension CoreDataFeedStore: Scheduler {
    @MainActor
    func schedule<T>(_ action: @escaping @Sendable () throws -> T) async rethrows -> T {
        if contextQueue == .main { return try action() }
        else { return try await perform(action) }
    }
}

// InMemory: execute immediately on main
extension InMemoryFeedStore: Scheduler {
    @MainActor
    func schedule<T>(_ action: @escaping @Sendable () throws -> T) async rethrows -> T {
        try action()
    }
}
```

---

## UI Composer — UIKit (`FeedUIComposer`)

Static factory that wires the presenter→adapter→view chain for a feature. Returns a fully configured view controller.

```swift
@MainActor
public final class FeedUIComposer {
    private init() {}  // Static factory — no instances

    public static func feedComposedWith(
        feedLoader: @MainActor @escaping () async throws -> Paginated<FeedImage>,
        imageLoader: @MainActor @escaping (URL) async throws -> Data,
        selection: @MainActor @escaping (FeedImage) -> Void = { _ in }
    ) -> ListViewController {
        let presentationAdapter = LoadResourcePresentationAdapter<Paginated<FeedImage>, FeedViewAdapter>(
            loader: feedLoader
        )

        let feedController = makeFeedViewController(title: FeedPresenter.title)
        feedController.onRefresh = presentationAdapter.loadResource

        presentationAdapter.presenter = LoadResourcePresenter(
            resourceView: FeedViewAdapter(
                controller: feedController,
                imageLoader: imageLoader,
                selection: selection),
            loadingView: WeakRefVirtualProxy(feedController),
            errorView: WeakRefVirtualProxy(feedController))

        return feedController
    }
}
```

**Wiring chain**: `ListViewController.onRefresh` → `PresentationAdapter.loadResource` → async loader → `Presenter.didFinishLoading` → `FeedViewAdapter.display` → `ListViewController.display([CellController]...)`

---

## UI Composer — SwiftUI Equivalent

```swift
@MainActor
struct FeedUIComposer {
    static func feedComposedWith(
        feedLoader: @escaping () async throws -> Paginated<FeedImage>,
        imageLoader: @escaping (URL) async throws -> Data,
        selection: @escaping (FeedImage) -> Void = { _ in }
    ) -> some View {
        let viewModel = FeedViewModel(loader: feedLoader, imageLoader: imageLoader)

        FeedView(viewModel: viewModel, selection: selection)
            .task { await viewModel.load() }
    }
}
```

In SwiftUI, `@Observable` replaces the adapter→proxy→presenter chain. The view model directly holds loading/error/success state and SwiftUI observes mutations automatically. No `WeakRefVirtualProxy` needed — SwiftUI manages view lifecycle.

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

**Key details**: `Task.immediate` ensures synchronous-first execution (important for avoiding unnecessary async hops). `[weak self]` prevents retain cycles. `isLoading` guard prevents duplicate requests. Cancellation on `deinit` cleans up in-flight tasks.

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
