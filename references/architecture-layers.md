# Architecture Layers — Code Reference

Use this when:
- You need to define a domain model, use-case protocol, or layer boundary
- You need to understand what imports are allowed in each layer
- You need code patterns for API mappers, cache layer, or presentation
- You need UIKit or SwiftUI UI layer patterns

Skip this file if:
- You need Composition Root wiring. Use `composition-root.md`.
- You need adapter/proxy patterns. Use `adapters-and-proxies.md`.
- You need concurrency annotations rationale. Use `concurrency-at-boundaries.md`.

Jump to:
- [Feature Layer](#feature-layer-foundation-only)
- [API Layer](#api-layer-foundation-only)
- [Cache Layer](#cache-layer-foundation-only)
- [Presentation Layer](#presentation-layer-foundation-only)
- [UI Layer — UIKit](#ui-layer--uikit)
- [UI Layer — SwiftUI](#ui-layer--swiftui-equivalent)

---

## Feature Layer (Foundation only)

Domain models are value types. Use-case contracts are protocols. No framework imports.

```swift
// Domain model — struct, Hashable, Sendable
public struct FeedImage: Hashable, Sendable {
    public let id: UUID
    public let description: String?
    public let location: String?
    public let url: URL
}

// Use-case protocol — cache boundary
public protocol FeedCache {
    func save(_ feed: [FeedImage]) throws
}

// Use-case protocol — data loading boundary
public protocol FeedImageDataLoader {
    func loadImageData(from url: URL) throws -> Data
}

// Use-case protocol — data caching boundary
public protocol FeedImageDataCache {
    func save(_ data: Data, for url: URL) throws
}

// Store protocol — image data persistence
public protocol FeedImageDataStore {
    func insert(_ data: Data, for url: URL) throws
    func retrieve(dataForURL url: URL) throws -> Data?
}
```

**Rules**: Models own no behavior. Protocols define single-responsibility boundaries. Each protocol maps to one use case.

```swift
// Correct: value type + Sendable + Hashable
public struct FeedImage: Hashable, Sendable {  // ...
}

// Incorrect: class, mutable, not Sendable
public class FeedImage {  // BAD — reference type, not thread-safe
    var id: UUID           // BAD — mutable
}
```

---

## API Layer (Foundation only)

Endpoint enums build URLs. Static mappers transform `Data + HTTPURLResponse` into domain models. Private `Decodable` types never leak.

```swift
// Endpoint — enum with associated values for pagination
public enum FeedEndpoint {
    case get(after: FeedImage? = nil)

    public func url(baseURL: URL) -> URL {
        switch self {
        case let .get(image):
            var components = URLComponents()
            components.scheme = baseURL.scheme
            components.host = baseURL.host
            components.path = baseURL.path + "/v1/feed"
            components.queryItems = [
                URLQueryItem(name: "limit", value: "10"),
                image.map { URLQueryItem(name: "after_id", value: $0.id.uuidString) },
            ].compactMap { $0 }
            return components.url!
        }
    }
}
```

```swift
// Static mapper — private Decodable types, public static function
public final class FeedItemsMapper {
    private struct Root: Decodable {
        private let items: [RemoteFeedItem]

        private struct RemoteFeedItem: Decodable {
            let id: UUID
            let description: String?
            let location: String?
            let image: URL
        }

        var images: [FeedImage] {
            items.map { FeedImage(id: $0.id, description: $0.description,
                                  location: $0.location, url: $0.image) }
        }
    }

    public enum Error: Swift.Error {
        case invalidData
    }

    public static func map(_ data: Data, from response: HTTPURLResponse) throws -> [FeedImage] {
        guard response.isOK,
              let root = try? JSONDecoder().decode(Root.self, from: data) else {
            throw Error.invalidData
        }
        return root.images
    }
}
```

```swift
// HTTPClient protocol — async/await, returns tuple
public protocol HTTPClient {
    func get(from url: URL) async throws -> (Data, HTTPURLResponse)
}
```

**Rules**: Mappers are `final class` with `static` methods — no instances. Decodable types are `private`. Errors are typed enums. `HTTPClient` is a protocol, not `URLSession` directly.

---

## Cache Layer (Foundation only)

Store protocols abstract persistence. Local models decouple cache representation from domain. Cache policy encapsulates business rules.

```swift
// Store protocol — synchronous throws API
public typealias CachedFeed = (feed: [LocalFeedImage], timestamp: Date)

public protocol FeedStore {
    func deleteCachedFeed() throws
    func insert(_ feed: [LocalFeedImage], timestamp: Date) throws
    func retrieve() throws -> CachedFeed?
}

// Local model — cache-layer representation
public struct LocalFeedImage: Equatable {
    public let id: UUID
    public let description: String?
    public let location: String?
    public let url: URL
}
```

```swift
// Use-case orchestrator — injects store + time for testability
public final class LocalFeedLoader {
    private let store: FeedStore
    private let currentDate: () -> Date

    public init(store: FeedStore, currentDate: @escaping () -> Date) {
        self.store = store
        self.currentDate = currentDate
    }
}

extension LocalFeedLoader: FeedCache {
    public func save(_ feed: [FeedImage]) throws {
        try store.deleteCachedFeed()
        try store.insert(feed.toLocal(), timestamp: currentDate())
    }
}

extension LocalFeedLoader {
    public func load() throws -> [FeedImage] {
        if let cache = try store.retrieve(),
           FeedCachePolicy.validate(cache.timestamp, against: currentDate()) {
            return cache.feed.toModels()
        }
        return []
    }
}
```

```swift
// Cache policy — encapsulated business rule, no instance state
final class FeedCachePolicy {
    private init() {}

    private static let calendar = Calendar(identifier: .gregorian)
    private static var maxCacheAgeInDays: Int { 7 }

    static func validate(_ timestamp: Date, against date: Date) -> Bool {
        guard let maxCacheAge = calendar.date(byAdding: .day,
                                               value: maxCacheAgeInDays,
                                               to: timestamp) else { return false }
        return date < maxCacheAge
    }
}
```

**Rules**: `LocalFeedLoader` converts between `FeedImage` <-> `LocalFeedImage` at the boundary. `currentDate` is injected as a closure for test control. Cache policy is a separate type — not embedded in the loader. Store protocol uses `throws` (not callbacks).

---

## Presentation Layer (Foundation only)

Generic presenter drives loading/error/success state. View protocols use associated types and `@MainActor`. View models are plain structs.

```swift
// View protocols — @MainActor for thread-safe UI updates
@MainActor
public protocol ResourceView {
    associatedtype ResourceViewModel
    func display(_ viewModel: ResourceViewModel)
}

@MainActor
public protocol ResourceLoadingView {
    func display(_ viewModel: ResourceLoadingViewModel)
}

@MainActor
public protocol ResourceErrorView {
    func display(_ viewModel: ResourceErrorViewModel)
}
```

`@MainActor` on view protocols is a **concurrency annotation**, not a framework dependency. The Presentation layer still imports only Foundation. This annotation ensures all display calls happen on the main thread.

```swift
// View models — plain structs
public struct ResourceLoadingViewModel {
    public let isLoading: Bool
}

public struct ResourceErrorViewModel {
    public let message: String?

    static var noError: ResourceErrorViewModel { .init(message: nil) }
    static func error(message: String) -> ResourceErrorViewModel { .init(message: message) }
}
```

```swift
// Generic presenter — reusable across any resource type
@MainActor
public final class LoadResourcePresenter<Resource, View: ResourceView> {
    public typealias Mapper = (Resource) throws -> View.ResourceViewModel

    private let resourceView: View
    private let loadingView: ResourceLoadingView
    private let errorView: ResourceErrorView
    private let mapper: Mapper

    public static var loadError: String {
        NSLocalizedString("GENERIC_CONNECTION_ERROR",
                          tableName: "Shared",
                          bundle: Bundle(for: Self.self),
                          comment: "Error message displayed when we can't load the resource from the server")
    }

    public init(resourceView: View, loadingView: ResourceLoadingView,
                errorView: ResourceErrorView, mapper: @escaping Mapper) {
        self.resourceView = resourceView
        self.loadingView = loadingView
        self.errorView = errorView
        self.mapper = mapper
    }

    // Convenience: when Resource == View.ResourceViewModel, mapper is identity
    public init(resourceView: View, loadingView: ResourceLoadingView,
                errorView: ResourceErrorView) where Resource == View.ResourceViewModel {
        self.resourceView = resourceView
        self.loadingView = loadingView
        self.errorView = errorView
        self.mapper = { $0 }
    }

    public func didStartLoading() {
        errorView.display(.noError)
        loadingView.display(ResourceLoadingViewModel(isLoading: true))
    }

    public func didFinishLoading(with resource: Resource) {
        do {
            resourceView.display(try mapper(resource))
            loadingView.display(ResourceLoadingViewModel(isLoading: false))
        } catch {
            didFinishLoading(with: error)
        }
    }

    public func didFinishLoading(with error: Error) {
        errorView.display(.error(message: Self.loadError))
        loadingView.display(ResourceLoadingViewModel(isLoading: false))
    }
}
```

**Rules**: Presenter has no UIKit/SwiftUI imports. Error strings use `NSLocalizedString` with `Bundle(for: Self.self)`. The `Mapper` typealias allows resource transformation (e.g., `Data -> UIImage`). Identity mapper convenience init eliminates boilerplate when types match.

---

## UI Layer — UIKit

`ListViewController` uses `DiffableDataSource`. `CellController` type-erases cell behavior. Views conform to presenter protocols.

```swift
// CellController — type-erased cell composition
public struct CellController {
    let id: any Hashable & Sendable
    let dataSource: UITableViewDataSource
    let delegate: UITableViewDelegate?
    let dataSourcePrefetching: UITableViewDataSourcePrefetching?

    public init(id: any Hashable & Sendable, _ dataSource: UITableViewDataSource) { ... }
}

extension CellController: Equatable, Hashable { /* hash/compare by id */ }
```

```swift
// ListViewController — DiffableDataSource + refresh + error view
public final class ListViewController: UITableViewController,
    UITableViewDataSourcePrefetching, ResourceLoadingView, ResourceErrorView {

    private(set) public var errorView = ErrorView()
    public var onRefresh: (() -> Void)?

    public func display(_ sections: [CellController]...) {
        var snapshot = NSDiffableDataSourceSnapshot<Int, CellController>()
        sections.enumerated().forEach { section, cellControllers in
            snapshot.appendSections([section])
            snapshot.appendItems(cellControllers, toSection: section)
        }
        if snapshot.itemIdentifiers != dataSource.snapshot().itemIdentifiers {
            dataSource.applySnapshotUsingReloadData(snapshot)
        }
    }
}
```

**Rules**: View controllers are thin — display logic only. `CellController` wraps any cell type through protocol composition. `onRefresh` closure decouples from specific loading logic. Multiple sections supported via variadic parameter.

---

## UI Layer — SwiftUI Equivalent

Same architectural boundaries, different binding mechanism:

```swift
// @Observable view model replaces WeakRefVirtualProxy + delegate pattern
@Observable
final class FeedViewModel {
    private(set) var items: [FeedImageViewModel] = []
    private(set) var isLoading = false
    private(set) var errorMessage: String?

    private let loader: () async throws -> Paginated<FeedImage>

    func load() async {
        isLoading = true
        errorMessage = nil
        do {
            let page = try await loader()
            items = page.items.map(FeedImagePresenter.map)
            isLoading = false
        } catch {
            errorMessage = LoadResourcePresenter<Any, AnyResourceView>.loadError
            isLoading = false
        }
    }
}

// SwiftUI View — thin, displays view model state
struct FeedView: View {
    @State var viewModel: FeedViewModel

    var body: some View {
        List(viewModel.items) { item in
            FeedImageRow(viewModel: item)
        }
        .refreshable { await viewModel.load() }
        .overlay { if viewModel.isLoading { ProgressView() } }
    }
}
```

**Rules**: `@Observable` replaces the `WeakRefVirtualProxy` pattern — SwiftUI manages view lifecycle. Composition Root creates the `@Observable` view model and passes it via `@State` or `.environment()`. Presenter logic (mapping, error strings) is still reused from the shared Presentation layer.

---

## Guardrails

- Do not import UIKit/SwiftUI in Feature, API, Cache, or Presentation layers — Foundation only
- Do not use `class` for domain models — use `struct` with `Hashable` and `Sendable`
- Do not embed cache policy in the loader — keep it as a separate type with `static` methods
- Do not leak `Decodable` types beyond the mapper — keep them `private`
- Do not add `@MainActor` to domain models or store protocols — isolation is decided at the composition level

## Verification

- [ ] Feature layer has zero framework imports beyond Foundation
- [ ] All domain models are `struct`, `Hashable`, `Sendable`
- [ ] All use-case boundaries are protocols (not concrete types)
- [ ] View protocols (`ResourceView`, `ResourceLoadingView`, `ResourceErrorView`) are `@MainActor`
- [ ] Mapper Decodable types are `private`
- [ ] `LocalFeedImage` is decoupled from `FeedImage` (separate type)
- [ ] Cache policy has no instance state — `static` methods only
