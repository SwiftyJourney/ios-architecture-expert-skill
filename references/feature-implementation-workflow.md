# Feature Implementation Workflow

Use this when:
- You are building a feature end-to-end from scratch
- You need a step-by-step checklist for the Essential Developer methodology
- You need to know the correct order of implementation (BDD -> domain -> API -> cache -> presentation -> UI -> composition -> tests)

Skip this file if:
- You need detailed code for a specific layer. Use `architecture-layers.md`.
- You need Composition Root wiring details. Use `composition-root.md`.
- You need testing patterns in depth. Use `testing-strategy.md`.

Jump to:
- [Step 1: BDD Specs](#step-1-define-bdd-specs)
- [Step 2: Domain Model](#step-2-create-domain-model-feature-layer)
- [Step 3: Concurrency Annotations](#step-3-add-concurrency-annotations)
- [Step 4: API Layer](#step-4-build-api-endpoint--mapper)
- [Step 5: Cache Layer](#step-5-build-cache-layer)
- [Step 6: Presentation Layer](#step-6-build-presentation-layer)
- [Step 7: UI Layer](#step-7-build-ui)
- [Step 8: Composition Root](#step-8-wire-in-composition-root)
- [Step 9: Acceptance Tests](#step-9-write-acceptance-tests)
- [Step 10: Snapshot Tests](#step-10-add-snapshot-tests)
- [Step 11: Concurrency Verification](#step-11-concurrency-verification)

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

## Step 3: Add Concurrency Annotations

After defining the domain model and use-case protocols, annotate for Swift 6 strict concurrency:

Checklist:
- [ ] All domain models are `Sendable` (automatic for value types with `Sendable` stored properties)
- [ ] View protocols (`ResourceView`, `ResourceLoadingView`, `ResourceErrorView`) are `@MainActor`
- [ ] Store protocols (`FeedStore`, `FeedImageDataStore`) use `throws` — no `@MainActor` (isolation is a composition decision)
- [ ] `HTTPClient` protocol uses `async throws` — no `@MainActor`

```swift
// Correct: view protocols are @MainActor
@MainActor
public protocol ResourceView {
    associatedtype ResourceViewModel
    func display(_ viewModel: ResourceViewModel)
}

// Correct: store protocols are NOT @MainActor
public protocol FeedStore {
    func deleteCachedFeed() throws
    func insert(_ feed: [LocalFeedImage], timestamp: Date) throws
    func retrieve() throws -> CachedFeed?
}
```

---

## Step 4: Build API Endpoint + Mapper

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

## Step 5: Build Cache Layer

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
- [ ] Conversion between domain <-> local models at the loader boundary
- [ ] `Scheduler` protocol awareness: store operations will be dispatched through `Scheduler.schedule` in the Composition Root
- [ ] InMemory fallback: ensure `InMemoryFeedStore` can serve as a production fallback
- [ ] Tests: save (delete-then-insert), load (with/without valid cache), validate

---

## Step 6: Build Presentation Layer

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
- [ ] Presenter is `@MainActor`
- [ ] View models are plain structs
- [ ] Error strings use `NSLocalizedString` with `Bundle(for:)`
- [ ] No UIKit/SwiftUI imports in presenter
- [ ] Feature-specific presenters use `static func map` (pure functions)
- [ ] Tests: verify all three state transitions (loading, success, error)
- [ ] Localization tests: verify all keys exist in all supported languages

---

## Step 7: Build UI

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

## Step 8: Wire in Composition Root

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
- [ ] Wire: adapter -> presenter -> view (+ proxy for UIKit)
- [ ] `loadMore` wired recursively through `FeedViewAdapter`
- [ ] Selection callback passed through for navigation
- [ ] `FeedService` uses `lazy var` for deferred dependency creation
- [ ] Store type constrained as `FeedStore & FeedImageDataStore & Scheduler & Sendable`
- [ ] Fallback strategy: CoreData -> InMemory on initialization failure
- [ ] `async let` used for parallel loading where applicable (e.g., `loadMoreRemoteFeed`)

---

## Step 9: Write Acceptance Tests

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

## Step 10: Add Snapshot Tests

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

---

## Step 11: Concurrency Verification

Final verification that all concurrency annotations are correct:

Checklist:
- [ ] Build with `SWIFT_STRICT_CONCURRENCY = complete` on all targets — zero warnings
- [ ] Run Thread Sanitizer (`-enableThreadSanitizer YES`) — zero data races
- [ ] Verify `@MainActor` is on: presenters, adapters, view controllers, composition code, view protocols
- [ ] Verify `@MainActor` is NOT on: domain models, use-case protocols, store protocols, cache policy
- [ ] Verify `@Sendable` on all closures crossing actor boundaries (especially `loadMore`, `Scheduler.schedule`)
- [ ] Verify `Task.immediate` is used in `LoadResourcePresentationAdapter` (not bare `Task`)
- [ ] Verify `Task.isCancelled` is checked after every `await` in adapter code

---

## Guardrails

- Do not skip BDD specs — they drive the acceptance tests in Step 9
- Do not jump to UI before domain and presentation are tested
- Do not wire dependencies outside the Composition Root
- Do not add concurrency annotations retroactively — Step 3 ensures they are baked in from the start

## Verification

- [ ] Every step has a passing test suite before moving to the next
- [ ] The feature decision tree in `SKILL.md` matches the workflow order
- [ ] All BDD scenarios from Step 1 are covered by acceptance tests in Step 9
