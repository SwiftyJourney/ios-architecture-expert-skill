# Feature Implementation Workflow

Step-by-step checklist for building a feature end-to-end following the Essential Developer methodology.

---

## Step 1: Define BDD Specs

Write a narrative and acceptance criteria before any code.

```
Narrative:
  As a customer
  I want to see a feed of images
  So I can browse interesting content

Scenario: Customer has connectivity
  Given the customer has connectivity
  When the customer requests to see the feed
  Then the app should display the latest feed from remote
  And replace the cache with the new feed

Scenario: Customer has no connectivity
  Given the customer has no connectivity
  And there is a cached version of the feed
  When the customer requests to see the feed
  Then the app should display the cached feed
```

---

## Step 2: Create Domain Model (Feature Layer)

```swift
// EssentialFeed/Feature/FeedImage.swift
public struct FeedImage: Hashable, Sendable {
    public let id: UUID
    public let description: String?
    public let location: String?
    public let url: URL
}
```

Checklist:
- [ ] `struct` (value type)
- [ ] `Hashable` (for diffable data sources / `ForEach`)
- [ ] `Sendable` (for async/await boundaries)
- [ ] Foundation only — no UIKit/SwiftUI imports
- [ ] Define use-case protocols: `FeedCache`, `FeedImageDataLoader`, etc.

---

## Step 3: Build API Endpoint + Mapper

```swift
// EssentialFeed/API/FeedEndpoint.swift
public enum FeedEndpoint {
    case get(after: FeedImage? = nil)
    public func url(baseURL: URL) -> URL { ... }
}

// EssentialFeed/API/FeedItemsMapper.swift
public final class FeedItemsMapper {
    private struct Root: Decodable { ... }
    public enum Error: Swift.Error { case invalidData }

    public static func map(_ data: Data, from response: HTTPURLResponse) throws -> [FeedImage] { ... }
}
```

Checklist:
- [ ] Endpoint is an `enum` with associated values
- [ ] Endpoint computes URL from a `baseURL` parameter
- [ ] Mapper is `static` — no instance state
- [ ] Decodable types are `private` (never leaked)
- [ ] Validates HTTP status code before decoding
- [ ] Tests: happy path, invalid data, non-2xx status codes

---

## Step 4: Build Cache Layer

```swift
// EssentialFeed/Cache/FeedStore.swift (protocol)
public protocol FeedStore {
    func deleteCachedFeed() throws
    func insert(_ feed: [LocalFeedImage], timestamp: Date) throws
    func retrieve() throws -> CachedFeed?
}

// EssentialFeed/Cache/LocalFeedImage.swift (cache model)
public struct LocalFeedImage: Equatable { ... }

// EssentialFeed/Cache/LocalFeedLoader.swift (orchestrator)
public final class LocalFeedLoader {
    public init(store: FeedStore, currentDate: @escaping () -> Date)
}

// EssentialFeed/Cache/FeedCachePolicy.swift (business rule)
final class FeedCachePolicy {
    static func validate(_ timestamp: Date, against date: Date) -> Bool
}
```

Checklist:
- [ ] Store protocol defines CRUD operations with `throws`
- [ ] Local model decoupled from domain model (separate `LocalFeedImage`)
- [ ] `currentDate` injected as closure for test control
- [ ] Cache policy is a separate type (not embedded in loader)
- [ ] Conversion between domain ↔ local models at the loader boundary
- [ ] Tests: save (delete-then-insert), load (with/without valid cache), validate

---

## Step 5: Build Presentation Layer

```swift
// EssentialFeed/Presentation/LoadResourcePresenter.swift
@MainActor
public final class LoadResourcePresenter<Resource, View: ResourceView> {
    public func didStartLoading()
    public func didFinishLoading(with resource: Resource)
    public func didFinishLoading(with error: Error)
}

// EssentialFeed/Presentation/FeedImagePresenter.swift
public final class FeedImagePresenter {
    public static func map(_ image: FeedImage) -> FeedImageViewModel { ... }
}

// Strings/Shared.strings
"GENERIC_CONNECTION_ERROR" = "Couldn't connect to server";
```

Checklist:
- [ ] Presenter is generic over `<Resource, View>`
- [ ] View models are plain structs
- [ ] Error strings use `NSLocalizedString` with `Bundle(for:)`
- [ ] No UIKit/SwiftUI imports in presenter
- [ ] Feature-specific presenters use `static func map` (pure functions)
- [ ] Tests: verify all three state transitions (loading, success, error)
- [ ] Localization tests: verify all keys exist in all supported languages

---

## Step 6: Build UI

### UIKit path:

```swift
// EssentialFeediOS/FeedImageCellController.swift
public final class FeedImageCellController: NSObject,
    UITableViewDataSource, UITableViewDelegate, UITableViewDataSourcePrefetching,
    ResourceView, ResourceLoadingView, ResourceErrorView { ... }
```

### SwiftUI path:

```swift
// EssentialFeediOS/FeedImageRow.swift
struct FeedImageRow: View {
    let viewModel: FeedImageViewModel
    @State var imageData: Data?
    var body: some View { ... }
}
```

Checklist:
- [ ] View conforms to `ResourceView`, `ResourceLoadingView`, `ResourceErrorView` (UIKit) or uses `@Observable` (SwiftUI)
- [ ] View displays view model data only — no business logic
- [ ] Cell reuse and cancellation handled (UIKit: prefetching delegate; SwiftUI: `.task` modifier)

---

## Step 7: Wire in Composition Root

```swift
// EssentialApp/FeedUIComposer.swift
public static func feedComposedWith(
    feedLoader: @escaping () async throws -> Paginated<FeedImage>,
    imageLoader: @escaping (URL) async throws -> Data,
    selection: @escaping (FeedImage) -> Void = { _ in }
) -> ListViewController { ... }
```

Checklist:
- [ ] Composer is a `static` factory — no stored state
- [ ] All closures are `@escaping` and `@MainActor` annotated
- [ ] Wire: adapter → presenter → view (+ proxy for UIKit)
- [ ] `loadMore` wired recursively through `FeedViewAdapter`
- [ ] Selection callback passed through for navigation

---

## Step 8: Write Acceptance Tests

```swift
func test_onLaunch_displaysRemoteFeedWhenCustomerHasConnectivity() async throws {
    let feed = try await launch(httpClient: .online(response), store: .empty)
    XCTAssertEqual(feed.numberOfRenderedFeedImageViews(), 2)
}
```

Checklist:
- [ ] Test each BDD scenario from Step 1
- [ ] Use `HTTPClientStub.online/.offline` for deterministic network
- [ ] Use `InMemoryFeedStore` for cross-launch cache persistence
- [ ] Inject via `SceneDelegate` convenience init
- [ ] DSL helper methods for readable assertions

---

## Step 9: Add Snapshot Tests

```swift
func test_feedWithContent() {
    let sut = makeSUT()
    sut.display(feedWithContent())
    assert(snapshot: sut.snapshot(for: .iPhone(style: .light)), named: "FEED_WITH_CONTENT_light")
    assert(snapshot: sut.snapshot(for: .iPhone(style: .dark)), named: "FEED_WITH_CONTENT_dark")
}
```

Checklist:
- [ ] Light mode + dark mode for every visual state
- [ ] At least one accessibility content size (e.g., `.extraExtraExtraLarge`)
- [ ] Use `ImageStub` or similar to provide controlled display data
- [ ] Test error states, loading states, empty states
- [ ] Record once, then assert on CI
