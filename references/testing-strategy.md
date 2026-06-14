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
- [Extracting a Protocol Boundary](#extracting-a-protocol-boundary)
- [Specification Pattern](#2-specification-pattern-shared-store-specs)
- [Integration Tests](#3-integration-tests)
- [HTTPClient Infrastructure Tests](#testing-the-httpclient-infrastructure-urlprotocol-stubbing)
- [End-to-End API Tests](#end-to-end-api-tests)
- [Snapshot Tests](#4-snapshot-tests)
- [Acceptance Tests](#5-acceptance-tests)
- [Generic Async Test Spy](#generic-async-test-spy-loaderspy)
- [Cross-Cutting Patterns](#cross-cutting-patterns)

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

**Swift Testing delta**: `struct` suites get a fresh instance per test, but `addTeardownBlock`/`trackForMemoryLeaks` is XCTest-only with no direct equivalent. For `@Test`/`#expect` syntax, defer to the `swift-testing-expert` skill.

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

Refinements:
- Capture completions instead of stubbing eager responses — the test decides when and how many times a result is delivered. This is what enables "does not deliver result after deallocation/cancellation" tests.
- Record messages in an `Array` when call ORDER is the contract (delete-then-insert); in a `Set` when order is incidental (presenter view messages), so reordering display calls doesn't break tests.
- Dealloc/cancellation test pattern: create the SUT as an optional, start the operation capturing results, nil the SUT (or cancel), THEN complete the spy, assert nothing was delivered. UIKit deinit tests need `autoreleasepool { }` around SUT creation or the assertion is flaky.

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

## Extracting a Protocol Boundary

When a test would require real infrastructure (network, database, a framework singleton), do not reach for the singleton or a subclass mock:

1. Define a minimal protocol with exactly the methods the SUT needs — shaped by the use case, never by the infrastructure's API surface.
2. Inject it via the initializer.
3. The abstraction lives next to the consumer module; the concrete infrastructure type conforms later, in its own module.

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
    // ... one assertThat helper per spec method
}
```

**Concrete test class** just calls the shared assertions:

```swift
// CoreData's failure modes can't be triggered deterministically — base spec only
class CoreDataFeedStoreTests: XCTestCase, FeedStoreSpecs {
    func test_retrieve_deliversEmptyOnEmptyCache() async throws {
        let sut = try makeSUT()
        assertThatRetrieveDeliversEmptyOnEmptyCache(on: sut)
    }
    // ... each spec method is one line delegating to the shared assertion

    // Return the PROTOCOL, not the concrete type — proves substitutability
    private func makeSUT() throws -> FeedStore { ... }
}
```

Conform to `Failable*` specs only when the implementation can deterministically simulate that failure (the since-deleted file-based store used invalid URLs/corrupt data). CoreData and InMemory conform to `FeedStoreSpecs` only.

**Workflow for a new store implementation**:
1. Prerequisite: one assertion per behavior per test — only then can specs be extracted.
2. Start with a test class conforming to the spec protocol with EMPTY method bodies — a compiler-enforced TODO list.
3. Fill one behavior per commit, each body one line delegating to the shared `assertThat` helper.

The same pattern repeats for `FeedImageDataStoreSpecs` — both the CoreData and InMemory stores conform.

---

## 3. Integration Tests

Test real infrastructure crossing module boundaries. Use actual CoreData stack, not mocks.

```swift
@MainActor
class CacheFeedIntegrationTests: XCTestCase {
    override func setUp() async throws {
        try await super.setUp()
        setupEmptyStoreState()      // clean BEFORE too — guards against a crashed previous run
    }

    override func tearDown() async throws {
        undoStoreSideEffects()
        try await super.tearDown()
    }

    // setupEmptyStoreState() and undoStoreSideEffects() both call deleteStoreArtifacts(),
    // which removes the file at testSpecificStoreURL()

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
        // named after the test class — parallel test classes never share artifacts
        cachesDirectory().appendingPathComponent("\(type(of: self)).store")
    }
}
```

**Purpose**: Verify that CoreData correctly persists and retrieves data across separate loader instances. Catches serialization bugs that unit tests with spies would miss.

---

## Testing the HTTPClient Infrastructure (URLProtocol Stubbing)

Never subclass `URLSession`, and never add mirror protocols (`HTTPSession`) whose only purpose is testability. Test the real `URLSession` path by registering a `URLProtocolStub: URLProtocol` whose `canInit` returns `true` for EVERY request — no test may hit the network.

```swift
class URLProtocolStub: URLProtocol {
    override class func canInit(with request: URLRequest) -> Bool { true } // intercept everything

    override func startLoading() {
        // 1. Deliver the stubbed (data, response, error) through `client`
        // 2. Call client?.urlProtocolDidFinishLoading(self) BEFORE invoking any request
        //    observer — otherwise URLSession's loading threads outlive the test
        //    (a data race that is invisible without TSan)
        // 3. Trigger request observers HERE (instance-level), never from class-level canInit
    }
}
```

- Register in `setUp`, unregister in `tearDown` (`URLProtocol.registerClass`/`unregisterClass`).
- Synchronize stub state behind a serial queue — URLProtocol callbacks race the test thread.
- Enumerate every invalid `(data, response, error)` combination in one table-driven `resultErrorFor` test; map them all to a single private `UnexpectedValuesRepresentation` error.
- Empty body is success: the URL Loading System delivers non-nil EMPTY `Data` for valid responses (e.g. 204) — do not require `data.count > 0`.
- The async `URLSession.data(from:)` API structurally cannot produce some invalid combinations — shrink the table when migrating rather than stubbing impossible states.

---

## End-to-End API Tests

Real client + loader composition against a backend test account with FIXED known data, asserted index-by-index:

- Separate target, excluded from the default dev scheme; run by a CI-only scheme.
- MUST use `URLSession(configuration: .ephemeral)` — the shared session's on-disk cache serves stale responses across runs (false greens).

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

In-process end-to-end tests that exercise the real `SceneDelegate` composition. `launch` is **synchronous** (`throws`, not `async`): determinism comes from `Task.immediate` adapters + an `HTTPClientStub` whose async `get` returns without ever suspending + a `CoreDataFeedStore` on `/dev/null` with `contextQueue: .main`, so the whole first load completes *before* `launch` returns. No sleeps.

```swift
@MainActor
class FeedAcceptanceTests: XCTestCase {
    func test_onLaunch_displaysRemoteFeedWhenCustomerHasConnectivity() throws {
        let store = try CoreDataFeedStore.empty
        let feed = try launch(httpClient: .online(response), store: store)

        XCTAssertEqual(feed.numberOfRenderedFeedImageViews(), 2)

        try store.withWaitingChanges { feed.simulateLoadMoreFeedAction() }

        XCTAssertEqual(feed.numberOfRenderedFeedImageViews(), 3)
    }

    func test_onLaunch_displaysCachedRemoteFeedWhenCustomerHasNoConnectivity() throws {
        let sharedStore = try CoreDataFeedStore.empty

        // First launch: cache remote data (same store instance carried across "launches")
        let onlineFeed = try launch(httpClient: .online(response), store: sharedStore)
        onlineFeed.simulateFeedImageViewVisible(at: 0)

        // Second launch: offline — shows cached data
        let offlineFeed = try launch(httpClient: .offline, store: sharedStore)
        XCTAssertEqual(offlineFeed.numberOfRenderedFeedImageViews(), 1)
    }

    private func launch(httpClient: HTTPClientStub = .offline,
                        store: CoreDataFeedStore) throws -> ListViewController {
        let sut = SceneDelegate(httpClient: httpClient, store: store)
        // UIWindow(frame:) is deprecated on iOS 26 — build a dummy scene (no public init exists)
        let scene = try XCTUnwrap((UIWindowScene.self as NSObject.Type).init() as? UIWindowScene)
        sut.window = UIWindow(windowScene: scene)
        sut.configureWindow()

        let nav = sut.window?.rootViewController as? UINavigationController
        let feed = nav?.topViewController as! ListViewController
        feed.simulateAppearance()
        return feed
    }
}
```

**Key techniques**:
- `SceneDelegate.init(httpClient:store:)` is the test-injection seam — constructor injection, not launch-argument back doors.
- `HTTPClientStub.online(response)` / `.offline` gives deterministic network behavior.
- Test-target factory extensions on the **real** store provide the fixtures: `.empty`, `.withExpiredFeedCache`, `.withNonExpiredFeedCache` are `CoreDataFeedStore` extensions over `/dev/null`. `InMemoryFeedStore` is **not** used here — it is the production fallback (see `composition-root.md`).

```swift
extension CoreDataFeedStore {
    static var empty: CoreDataFeedStore {
        get throws {
            try CoreDataFeedStore(storeURL: URL(fileURLWithPath: "/dev/null"), contextQueue: .main)
        }
    }
    static var withExpiredFeedCache: CoreDataFeedStore {
        get throws {
            let store = try CoreDataFeedStore.empty
            try store.insert([], timestamp: .distantPast)
            return store
        }
    }
}
```

### Waiting for genuinely-parallel side effects — never `Task.sleep`

`launch` is synchronous, but actions like load-more use `async let` to fetch the remote page and the cache in parallel, so the resulting cache *save* lands after the action returns. Don't sleep — **poll for the observable state change**:

```swift
extension CoreDataFeedStore {
    func withWaitingChanges(_ action: () -> Void, timeout: TimeInterval = 1) throws {
        let state = try retrieve()?.timestamp
        action()
        let maxDate = Date() + timeout
        while Date() <= maxDate {
            if try retrieve()?.timestamp != state { return }   // the save became visible
            RunLoop.current.run(until: Date())                 // pump the run loop
        }
        throw Timeout()
    }
}
```

> Guardrail: never use `Task.sleep` to wait for loading in acceptance tests — `Task.immediate` + a main-context store makes launch synchronous; for genuinely-parallel side effects, poll for the state change with a timeout.

**Avoid XCUITest acceptance tests with launch-argument/`#if DEBUG` back doors.** The case study built exactly that — a `DebuggingSceneDelegate` subclass, a `-reset` argument, a `connectivity` UserDefault — and then deleted the entire UI-test target in favor of these in-process tests. Constructor injection at the composition root replaces every test hook, and no test code ships in the production `SceneDelegate`. (When you *do* have genuinely debug-only code — a debug menu, a simulator seam — the `#if DEBUG`-isolated `SceneDelegate` subclass is the right way to keep it out of release; see the "Debug-only code" pattern in `composition-root.md`.)

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

`makeSUT` wires the cleanup so streams can't leak across tests, and cancellation tests must `await` the `.cancelled` result before asserting cancelled URLs (async cancellation isn't synchronous like Combine's `receiveCancel`):

```swift
addTeardownBlock { [weak loader] in try await loader?.cancelPendingRequests() }
```

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

## Cross-Cutting Patterns

### Domain-named test DSL for business rules

Hide business-rule constants behind domain-named helpers so tests assert *semantics*, not the current value. If the cache max-age changes from 7 days, only the helper changes:

```swift
extension Date {
    func minusFeedCacheMaxAge() -> Date { adding(days: -feedCacheMaxAgeInDays) }
}
```

Sample the boundary: test at expiration, and ±1 second around it (`minusFeedCacheMaxAge().adding(seconds: 1)` and `.adding(seconds: -1)`).

### Test mappers in isolation

Once loaders become composed `() async throws -> Resource` closures, the mapping logic is a static function — test it directly, with no `HTTPClient` double:

```swift
let result = try ImageCommentsMapper.map(json, from: HTTPURLResponse(statusCode: 200))
```

Assert thrown errors and mapped values per status code and payload. These replaced spy-based use-case tests when the loader protocols were deleted.

### Localization guard test

Iterate every supported localization and assert no key is missing a translation:

```swift
func test_localizedStrings_haveKeysAndValuesForAllSupportedLocalizations() {
    let bundle = Bundle(for: FeedPresenter.self)
    let table = "Feed"
    for localization in bundle.localizations {
        let localizedBundle = Bundle(path: bundle.path(forResource: localization, ofType: "lproj")!)!
        for key in keys(in: localizedBundle, table) {
            let value = localizedBundle.localizedString(forKey: key, value: nil, table: table)
            XCTAssertNotEqual(value, key, "Missing \(localization) translation for '\(key)'")
        }
    }
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
