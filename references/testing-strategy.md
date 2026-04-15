# Testing Strategy — Five Layers

Use this when:
- You need to write tests for a use case, store implementation, or UI composition
- You need the `makeSUT()` pattern, spy collaborators, or memory leak tracking
- You need the specification pattern for shared store contracts
- You need the generic async test spy (`LoaderSpy`)
- You need snapshot or acceptance test patterns

Skip this file if:
- You need general Swift Testing patterns (`@Test`, `#expect`). Use the `swift-testing-expert` skill.
- You need architecture layer details. Use `architecture-layers.md`.

Jump to:
- [Unit Tests](#1-unit-tests)
- [Specification Pattern](#2-specification-pattern-shared-store-specs)
- [Integration Tests](#3-integration-tests)
- [Snapshot Tests](#4-snapshot-tests)
- [Acceptance Tests](#5-acceptance-tests)
- [Generic Async Test Spy](#generic-async-test-spy-loaderspy)

---

## 1. Unit Tests

Every use case gets its own test class named by behavior. Tests follow a strict structure: `makeSUT()` factory, spy collaborators, `expect` helpers.

### `makeSUT()` + `trackForMemoryLeaks`

```swift
// XCTest version
@MainActor
class CacheFeedUseCaseTests: XCTestCase {
    func test_save_requestsNewCacheInsertionWithTimestampOnSuccessfulDeletion() {
        let timestamp = Date()
        let feed = uniqueImageFeed()
        let (sut, store) = makeSUT(currentDate: { timestamp })

        try? sut.save(feed.models)

        XCTAssertEqual(store.receivedMessages,
                       [.deleteCachedFeed, .insert(feed.local, timestamp)])
    }

    private func makeSUT(currentDate: @escaping () -> Date = Date.init,
                         file: StaticString = #filePath,
                         line: UInt = #line) -> (sut: LocalFeedLoader, store: FeedStoreSpy) {
        let store = FeedStoreSpy()
        let sut = LocalFeedLoader(store: store, currentDate: currentDate)
        trackForMemoryLeaks(store, file: file, line: line)
        trackForMemoryLeaks(sut, file: file, line: line)
        return (sut, store)
    }
}
```

```swift
// Swift Testing equivalent
@Suite @MainActor
struct CacheFeedUseCaseTests {
    @Test func save_requestsNewCacheInsertionWithTimestampOnSuccessfulDeletion() {
        let timestamp = Date()
        let feed = uniqueImageFeed()
        let (sut, store) = makeSUT(currentDate: { timestamp })

        try? sut.save(feed.models)

        #expect(store.receivedMessages == [.deleteCachedFeed, .insert(feed.local, timestamp)])
    }

    private func makeSUT(currentDate: @escaping () -> Date = Date.init) -> (sut: LocalFeedLoader, store: FeedStoreSpy) {
        let store = FeedStoreSpy()
        let sut = LocalFeedLoader(store: store, currentDate: currentDate)
        return (sut, store)
    }
}
```

### Spy with `receivedMessages` enum

```swift
class FeedStoreSpy: FeedStore {
    enum ReceivedMessage: Equatable {
        case deleteCachedFeed
        case insert([LocalFeedImage], Date)
        case retrieve
    }

    private(set) var receivedMessages = [ReceivedMessage]()
    private var deletionError: Error?
    private var insertionError: Error?
    private var retrievalResult: Result<CachedFeed?, Error>?

    func deleteCachedFeed() throws {
        receivedMessages.append(.deleteCachedFeed)
        if let error = deletionError { throw error }
    }

    func insert(_ feed: [LocalFeedImage], timestamp: Date) throws {
        receivedMessages.append(.insert(feed, timestamp))
        if let error = insertionError { throw error }
    }

    func retrieve() throws -> CachedFeed? {
        receivedMessages.append(.retrieve)
        return try retrievalResult?.get()
    }

    // Stubbing helpers
    func completeRetrieval(with feed: [LocalFeedImage], timestamp: Date) {
        retrievalResult = .success(CachedFeed(feed: feed, timestamp: timestamp))
    }
}
```

**Pattern**: Enum-based message recording captures method calls AND arguments. Separate stub methods configure return values. This keeps assertions on **what happened** separate from **how to respond**.

### Common test helpers

```swift
func anyURL() -> URL { URL(string: "http://any-url.com")! }
func anyNSError() -> NSError { NSError(domain: "any error", code: 0) }
func anyData() -> Data { Data("any data".utf8) }

func uniqueImage() -> FeedImage {
    FeedImage(id: UUID(), description: "any", location: "any", url: anyURL())
}

func uniqueImageFeed() -> (models: [FeedImage], local: [LocalFeedImage]) {
    let models = [uniqueImage(), uniqueImage()]
    let local = models.map { LocalFeedImage(id: $0.id, description: $0.description,
                                             location: $0.location, url: $0.url) }
    return (models, local)
}
```

### Memory leak tracking (XCTest)

```swift
extension XCTestCase {
    func trackForMemoryLeaks(_ instance: AnyObject,
                             file: StaticString = #filePath,
                             line: UInt = #line) {
        addTeardownBlock { [weak instance] in
            XCTAssertNil(instance, "Instance should have been deallocated. Potential memory leak.",
                         file: file, line: line)
        }
    }
}
```

---

## 2. Specification Pattern (Shared Store Specs)

Protocol-driven test specifications define the **contract** that any `FeedStore` implementation must satisfy. Each implementation runs the same specs.

```swift
// Specification protocol — defines required test methods
protocol FeedStoreSpecs {
    func test_retrieve_deliversEmptyOnEmptyCache() async throws
    func test_retrieve_hasNoSideEffectsOnEmptyCache() async throws
    func test_retrieve_deliversFoundValuesOnNonEmptyCache() async throws
    func test_insert_overridesPreviouslyInsertedCacheValues() async throws
    func test_delete_emptiesPreviouslyInsertedCache() async throws
}

protocol FailableRetrieveFeedStoreSpecs: FeedStoreSpecs {
    func test_retrieve_deliversFailureOnRetrievalError() async throws
    func test_retrieve_hasNoSideEffectsOnFailure() async throws
}

protocol FailableInsertFeedStoreSpecs: FeedStoreSpecs {
    func test_insert_deliversErrorOnInsertionError() async throws
    func test_insert_hasNoSideEffectsOnInsertionError() async throws
}

protocol FailableDeleteFeedStoreSpecs: FeedStoreSpecs {
    func test_delete_deliversErrorOnDeletionError() async throws
    func test_delete_hasNoSideEffectsOnDeletionError() async throws
}

// Composite protocol — all failable specs
typealias FailableFeedStoreSpecs = FailableRetrieveFeedStoreSpecs
    & FailableInsertFeedStoreSpecs
    & FailableDeleteFeedStoreSpecs
```

### Shared assertion functions (free functions in extensions)

```swift
extension FeedStoreSpecs where Self: XCTestCase {
    func assertThatRetrieveDeliversEmptyOnEmptyCache(
        on sut: FeedStore,
        file: StaticString = #filePath,
        line: UInt = #line
    ) {
        expect(sut, toRetrieve: .success(.none), file: file, line: line)
    }

    func assertThatInsertOverridesPreviouslyInsertedCacheValues(
        on sut: FeedStore,
        file: StaticString = #filePath,
        line: UInt = #line
    ) {
        let firstFeed = uniqueImageFeed().local
        let firstTimestamp = Date()
        let latestFeed = uniqueImageFeed().local
        let latestTimestamp = Date()

        insert((firstFeed, firstTimestamp), to: sut)
        insert((latestFeed, latestTimestamp), to: sut)

        expect(sut, toRetrieve: .success(CachedFeed(feed: latestFeed,
                                                     timestamp: latestTimestamp)),
               file: file, line: line)
    }
}
```

**Concrete test class** just calls the shared assertions:

```swift
// CoreData implements the full failable spec
class CoreDataFeedStoreTests: XCTestCase, FailableFeedStoreSpecs {
    func test_retrieve_deliversEmptyOnEmptyCache() async throws {
        let sut = try makeSUT()
        assertThatRetrieveDeliversEmptyOnEmptyCache(on: sut)
    }
    // ... each spec method delegates to shared assertion
}

// InMemory implements only the base spec (no failure modes)
class InMemoryFeedStoreTests: XCTestCase, FeedStoreSpecs {
    func test_retrieve_deliversEmptyOnEmptyCache() async throws {
        let sut = InMemoryFeedStore()
        assertThatRetrieveDeliversEmptyOnEmptyCache(on: sut)
    }
    // ... each spec method delegates to shared assertion
}
```

**Benefit**: Add a new `FeedStore` implementation (e.g., `RealmFeedStore`) -> conform to `FeedStoreSpecs` -> get all contract tests for free.

---

## 3. Integration Tests

Test real infrastructure crossing module boundaries. Use actual CoreData stack, not mocks.

```swift
@MainActor
class CacheFeedIntegrationTests: XCTestCase {
    func test_load_deliversItemsSavedOnASeparateInstance() throws {
        let storeURL = testSpecificStoreURL()
        let sutToSave = try makeSUT(storeURL: storeURL)
        let sutToLoad = try makeSUT(storeURL: storeURL)
        let feed = uniqueImageFeed().models

        try sutToSave.save(feed)
        let loaded = try sutToLoad.load()

        XCTAssertEqual(loaded, feed)
    }

    private func makeSUT(storeURL: URL) throws -> LocalFeedLoader {
        let store = try CoreDataFeedStore(storeURL: storeURL)
        return LocalFeedLoader(store: store, currentDate: Date.init)
    }

    private func testSpecificStoreURL() -> URL {
        cachesDirectory().appendingPathComponent("\(type(of: self)).store")
    }
}
```

**Purpose**: Verify that CoreData correctly persists and retrieves data across separate loader instances. Catches serialization bugs that unit tests with spies would miss.

---

## 4. Snapshot Tests

Visual regression testing across light/dark modes and accessibility sizes.

```swift
@MainActor
class FeedSnapshotTests: XCTestCase {
    func test_feedWithContent() {
        let sut = makeSUT()

        sut.display(feedWithContent())

        assert(snapshot: sut.snapshot(for: .iPhone(style: .light)), named: "FEED_WITH_CONTENT_light")
        assert(snapshot: sut.snapshot(for: .iPhone(style: .dark)), named: "FEED_WITH_CONTENT_dark")
        assert(snapshot: sut.snapshot(for: .iPhone(style: .light, contentSize: .extraExtraExtraLarge)),
               named: "FEED_WITH_CONTENT_light_extraExtraExtraLarge")
    }

    func test_feedWithFailedImageLoading() {
        let sut = makeSUT()
        sut.display(feedWithFailedImageLoading())
        assert(snapshot: sut.snapshot(for: .iPhone(style: .light)), named: "FEED_WITH_FAILED_IMAGE_light")
        assert(snapshot: sut.snapshot(for: .iPhone(style: .dark)), named: "FEED_WITH_FAILED_IMAGE_dark")
    }
}
```

**Test matrix**: Each visual state x (light, dark) x accessibility sizes. Use `ImageStub` helpers to provide controlled display data without network calls.

---

## 5. Acceptance Tests

End-to-end tests with the full DI container. Use `HTTPClientStub` for deterministic network behavior.

```swift
@MainActor
class FeedAcceptanceTests: XCTestCase {
    func test_onLaunch_displaysRemoteFeedWhenCustomerHasConnectivity() async throws {
        let feed = try await launch(httpClient: .online(response), store: .empty)

        XCTAssertEqual(feed.numberOfRenderedFeedImageViews(), 2)
        XCTAssertEqual(feed.renderedFeedImageData(at: 0), makeImageData0())
        XCTAssertEqual(feed.renderedFeedImageData(at: 1), makeImageData1())
    }

    func test_onLaunch_displaysCachedRemoteFeedWhenCustomerHasNoConnectivity() async throws {
        // First launch: cache remote data
        let onlineFeed = try await launch(httpClient: .online(response), store: .empty)
        onlineFeed.simulateFeedImageViewVisible(at: 0)

        // Second launch: offline — shows cached data
        let offlineFeed = try await launch(httpClient: .offline, store: onlineFeed.store)
        XCTAssertEqual(offlineFeed.numberOfRenderedFeedImageViews(), 2)
    }

    private func launch(httpClient: HTTPClientStub,
                        store: InMemoryFeedStore) async throws -> ListViewController {
        let sut = SceneDelegate(httpClient: httpClient, store: store)
        sut.window = UIWindow(frame: CGRect(x: 0, y: 0, width: 1, height: 1))
        sut.configureWindow()

        let nav = sut.window?.rootViewController as? UINavigationController
        let feed = nav?.topViewController as! ListViewController
        feed.simulateAppearance()
        // Allow async loading
        try await Task.sleep(nanoseconds: 100_000_000)
        return feed
    }
}
```

**Key techniques**:
- `SceneDelegate` has a convenience init accepting `HTTPClient` + store for test injection
- `HTTPClientStub.online(response)` / `.offline` provides deterministic network behavior
- `InMemoryFeedStore` persists across "launches" to test cache behavior
- DSL methods (`numberOfRenderedFeedImageViews`, `renderedFeedImageData`) keep assertions readable

### HTTPClientStub patterns

```swift
class HTTPClientStub: HTTPClient {
    private let stub: (URL) -> Result<(Data, HTTPURLResponse), Error>

    static var offline: HTTPClientStub {
        HTTPClientStub { _ in .failure(NSError(domain: "offline", code: 0)) }
    }

    static func online(_ stub: @escaping (URL) -> (Data, HTTPURLResponse)) -> HTTPClientStub {
        HTTPClientStub { url in .success(stub(url)) }
    }

    func get(from url: URL) async throws -> (Data, HTTPURLResponse) {
        try stub(url).get()
    }
}
```

### InMemoryFeedStore factory extensions

```swift
extension InMemoryFeedStore {
    static var empty: InMemoryFeedStore { InMemoryFeedStore() }
}
```

The same `InMemoryFeedStore` instance is passed between simulated "launches" to verify cache persistence across app lifecycle events.

---

## Generic Async Test Spy (`LoaderSpy`)

A generic, reusable test spy for async loaders using `AsyncThrowingStream`. Used in UI integration tests to control async completion timing.

```swift
@MainActor
class LoaderSpy<Param, Resource: Sendable> {
    private(set) var requests = [(
        param: Param,
        stream: AsyncThrowingStream<Resource, Error>,
        continuation: AsyncThrowingStream<Resource, Error>.Continuation,
        result: AsyncResult?
    )]()

    private struct NoResponse: Error {}
    private struct Timeout: Error {}

    func load(_ param: Param) async throws -> Resource {
        let (stream, continuation) = AsyncThrowingStream<Resource, Error>.makeStream()
        let index = requests.count
        requests.append((param, stream, continuation, nil))

        do {
            for try await result in stream {
                try Task.checkCancellation()
                requests[index].result = .success
                return result
            }

            try Task.checkCancellation()

            throw NoResponse()
        } catch {
            requests[index].result = Task.isCancelled ? .cancelled : .failure
            throw error
        }
    }

    func complete(with resource: Resource, at index: Int) async {
        requests[index].continuation.yield(resource)
        requests[index].continuation.finish()

        while requests[index].result == nil { await Task.yield() }
    }

    func fail(with error: Error, at index: Int) async {
        requests[index].continuation.finish(throwing: error)

        while requests[index].result == nil { await Task.yield() }
    }

    func result(at index: Int, timeout: TimeInterval = 1) async throws -> AsyncResult {
        let maxDate = Date() + timeout

        while Date() <= maxDate {
            if let result = requests[index].result {
                return result
            }

            await Task.yield()
        }

        throw Timeout()
    }

    func cancelPendingRequests() async throws {
        for (index, request) in requests.enumerated() where request.result == nil {
            request.continuation.finish(throwing: CancellationError())

            while requests[index].result == nil { await Task.yield() }
        }
    }
}

enum AsyncResult {
    case success
    case failure
    case cancelled
}
```

**Key features**:
- `@MainActor` — matches the isolation of UI integration tests
- `AsyncThrowingStream` — gives test control over when results arrive
- `complete(with:at:)` and `fail(with:at:)` — test drives async completion
- `while result == nil { await Task.yield() }` — waits for the async consumer to process the result
- `result(at:timeout:)` — asserts the outcome with a timeout guard
- `cancelPendingRequests()` — cleans up in-flight streams

### Usage in UI integration tests

```swift
// Specialized spy for feed loading
typealias FeedLoaderSpy = LoaderSpy<Void, Paginated<FeedImage>>
typealias ImageLoaderSpy = LoaderSpy<URL, Data>

// In test
func test_loadFeedCompletion_rendersSuccessfullyLoadedFeed() async {
    let (sut, loader) = makeSUT()

    sut.simulateAppearance()
    await loader.complete(with: Paginated(items: [image0, image1]), at: 0)

    XCTAssertEqual(sut.numberOfRenderedFeedImageViews(), 2)
}
```

---

## Guardrails

- Use `makeSUT()` in every test class — never share a `sut` across tests
- Use enum-based message recording in spies — not boolean flags
- Use `trackForMemoryLeaks` in XCTest for all reference-type SUTs and collaborators
- Name test classes by behavior (`CacheFeedUseCaseTests`), not by implementation (`LocalFeedLoaderTests`)
- Do not mock infrastructure in integration tests — use the real CoreData stack

## Verification

- [ ] Every test class has a `makeSUT()` factory method
- [ ] Every `makeSUT()` in XCTest calls `trackForMemoryLeaks` on reference types
- [ ] Spy collaborators use `enum ReceivedMessage` for method recording
- [ ] Store implementations conform to the appropriate `FeedStoreSpecs` protocol
- [ ] Integration tests use separate `sutToSave` and `sutToLoad` instances
- [ ] Acceptance tests inject dependencies via `SceneDelegate` convenience init
- [ ] Snapshot tests cover light mode, dark mode, and accessibility sizes
