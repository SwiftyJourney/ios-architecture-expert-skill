# Testing Strategy — Five Layers

Use this when:
- You need to write tests for a use case, store implementation, or UI composition
- You need the `makeSUT()` pattern, spy collaborators, or memory leak tracking
- You need the specification pattern for shared store contracts
- You need the generic async test spy (`LoaderSpy`)
- You need snapshot or acceptance test patterns

Skip this file if:
- You need architecture layer details. Use `architecture-layers.md`.
- You need the reasoning behind a boundary. Use `design-principles.md`.

> **Scope note.** The sections that are genuinely *architecture-coupled* — testing through the
> public API, extracting a protocol boundary, contract specs, and acceptance tests through the
> Composition Root — are the ones this skill owns, because each one is a design decision wearing a
> test's clothes. The rest (double taxonomy, DSL craft, snapshot mechanics, TDD process, Swift
> Testing syntax) is planned to move to a dedicated testing skill; until that exists, it stays here
> so this file remains self-sufficient.

Jump to:
- [How much to test at which level](#how-much-to-test-at-which-level)
- [Test Through the Public API](#0-test-through-the-public-api-no-testable-import)
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

## How much to test at which level

The five layers below are not equal. The distribution is deliberate:

| Level | Volume | What it covers | Why not more |
|---|---|---|---|
| **Unit** | The overwhelming majority | Every edge case, every error course | Fast enough to run several times a minute |
| **Integration** | Some | The **happy path** of real components collaborating | Edge cases grow *exponentially* with the number of participants — and you cannot force most errors with real infrastructure |
| **Snapshot** | A few per visual state | Rendering, light/dark, large content sizes | Slower; catches regressions no unit test can see |
| **Acceptance** | One per BDD scenario | The composed app, launch and state transitions | The most valuable per test, the most expensive to keep passing |
| **UI (XCUITest)** | Ideally none | — | ~10s *per test*, stateful across launches, and it forces test code into the production target |

Two rules that follow:

- **Test edge cases in isolation; test only the happy path in integration.** Once each component's
  failure modes are proven at the unit level, integrating them only needs to prove they connect.
- **Test the least amount of detail at the highest level.** The more detail a high-level test
  asserts, the more fragile and slow it becomes. Details belong in unit tests.

And one rule specific to high-level tests:

> **Replace only infrastructure — never business logic.** Stub the HTTP client and point the store
> at `/dev/null`; do not substitute a use case, presenter or policy. That is what preserves the
> confidence that you are testing your real application.

> **Why this codebase has no XCUITest target.** It had one — with launch arguments, a
> `connectivity` `UserDefault`, and a `DebuggingSceneDelegate` — and deleted it. The in-process
> acceptance tests below replaced 3 UI tests taking ~30s with 5 integration tests taking ~0.2s,
> covered a use case the UI tests could not reach (cache validation on backgrounding), asserted
> *content* rather than mere existence of cells, and let every seam that had been opened for
> testing become private again. See [§5](#5-acceptance-tests).

---

## 0. Test Through the Public API (No `@testable import`)

**Tests exercise a module only through its public surface.** Test targets use a plain `import Feed` — never `@testable import Feed`. This is not a stylistic preference; it is a design constraint that keeps the tests honest:

- `@testable import` grants access to `internal` members, which lets tests couple to implementation details. When they do, refactoring internals breaks tests even though behavior is unchanged — the tests stop being a safety net and become friction.
- Testing through the public API forces the module to *have* a usable public API. If a behavior is hard to assert from outside, the design — not the test — is telling you something: a missing protocol boundary, a return value that should be exposed, or a collaborator that should be injected.

```swift
// Test target — plain import, exercises FeedAPI's public surface only
import XCTest
import FeedAPI          // NOT @testable import FeedAPI

class LoadFeedFromRemoteUseCaseTests: XCTestCase { ... }
```

### When a test can't reach what it needs

Do **not** widen the test's reach with `@testable`. Instead, make the behavior observable through the public API:

| You want to assert... | Public seam to expose it through |
|---|---|
| A collaborator was called with certain arguments | Inject a protocol; assert on a public spy conforming to it (`FeedStoreSpy: FeedStore`) |
| A produced value | Return it from a public method / deliver it via a public completion or `async` result |
| A mapped/derived result | Make the mapper a public static function and test it directly (`FeedItemsMapper.map(_:from:)`) |
| A composed graph behaves end-to-end | Assert through the composed public seam (e.g. `SceneDelegate.init(httpClient:store:)` → rendered view state), not private fields |

**Access-control implication**: production types, initializers, and methods that a test needs to reach must be declared `public` deliberately — that declaration *is* the module's contract. Test-only helpers and fixtures (spies, `makeSUT`, `.empty` store factories) live in the **test target**, so they need no production access at all.

**Same-target tests** (acceptance tests that live in the app target alongside `SceneDelegate`) don't need `import` at all, but the rule still holds in spirit: assert through the composed public seams — the injected `SceneDelegate` initializer and the rendered view — never private state.

### The one exception: the Composition Root

The rule exists to stop tests coupling to a module's internals, so that refactoring internals does
not break tests. **A root module has no external clients**, so there is no contract to rot:

- No other module references the Composition Root, so its types never need to be `public`.
- Declaring them `public` purely so a test can reach them would be inventing an API nobody uses.

So if the root lives in its own target, `@testable import App` is the correct tool there — and
*only* there. In every feature module, a test that needs `@testable` is still a missing public
seam. See `composition-root.md`.

> Guardrail: outside the Composition Root, if a test only compiles with `@testable`, treat it as a missing public seam, not a reason to keep `@testable`.

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

### Swift Testing delta

Most of this translates mechanically: `XCTestCase` → a `struct` or `final class` suite (a fresh
instance per test either way), `setUp`/`tearDown` → `init`/`deinit`, `XCTAssertEqual` → `#expect`,
`XCTUnwrap` → `try #require`, `XCTestExpectation` → `confirmation`, the `samples.enumerated()`
loop → a parameterized `@Test(arguments:)`, and `file:`/`line:` propagation in helpers →
a defaulted `SourceLocation` parameter forwarded to `#expect(..., sourceLocation:)`.

Two things do **not** translate mechanically:

- **Tests run in parallel by default.** Anything that shares state across tests — a store URL, an
  on-disk artifact, a registered `URLProtocol` stub — needs `.serialized` on the suite, or it will
  flake in ways the XCTest version never did. (Randomized-order discipline already prepared you
  for this; parallelism just makes it mandatory.)
- **`addTeardownBlock` has no direct equivalent**, so `trackForMemoryLeaks` must be rebuilt — see
  below.

### `trackForMemoryLeaks` in Swift Testing

The mechanism is a **test-scoping trait** (swift-evolution ST-0007, Swift 6.1). `provideScope`
wraps the test body, so it can assert *after* the body's locals have been released — which is
exactly what `addTeardownBlock` gave you. A task-local carries the per-test collector, so the
`trackForMemoryLeaks` call site stays as short as the XCTest one.

```swift
import Testing
import Foundation

private final class TrackedInstances: @unchecked Sendable {
    private struct Entry { weak var instance: AnyObject?; let location: SourceLocation }
    private let lock = NSLock()
    private var entries = [Entry]()

    func track(_ instance: AnyObject, _ location: SourceLocation) {
        lock.lock(); defer { lock.unlock() }
        entries.append(Entry(instance: instance, location: location))
    }

    func verify() {
        lock.lock(); defer { lock.unlock() }
        for entry in entries where entry.instance != nil {
            Issue.record("Instance should have been deallocated. Potential memory leak.",
                         sourceLocation: entry.location)
        }
    }
}

private enum LeakTracking {
    @TaskLocal static var current: TrackedInstances?
}

struct TrackLeaksTrait: TestTrait, SuiteTrait, TestScoping {
    var isRecursive: Bool { true }      // applies to each test, so failures attribute per test

    func provideScope(for test: Test,
                      testCase: Test.Case?,
                      performing function: @Sendable () async throws -> Void) async throws {
        let tracked = TrackedInstances()
        try await LeakTracking.$current.withValue(tracked) { try await function() }
        tracked.verify()                // the body's locals are released by now
    }
}

extension Trait where Self == TrackLeaksTrait {
    static var trackLeaks: Self { Self() }
}

func trackForMemoryLeaks(_ instance: AnyObject,
                         sourceLocation: SourceLocation = #_sourceLocation) {
    LeakTracking.current?.track(instance, sourceLocation)
}
```

Declare it once on the suite; `makeSUT` is otherwise unchanged:

```swift
@Suite(.trackLeaks)
struct CacheFeedUseCaseTests {
    @Test func save_requestsCacheDeletion() { let (sut, store) = makeSUT(); ... }

    private func makeSUT(currentDate: @escaping () -> Date = Date.init)
        -> (sut: LocalFeedLoader, store: FeedStoreSpy) {
        let store = FeedStoreSpy()
        let sut = LocalFeedLoader(store: store, currentDate: currentDate)
        trackForMemoryLeaks(store)
        trackForMemoryLeaks(sut)
        return (sut, store)
    }
}
```

Two details that matter:

- **`isRecursive` is what gives per-test attribution.** Without it, a suite-level trait wraps the
  whole suite and every leak is reported against the *suite* rather than the test that caused it.
- **`#_sourceLocation`** is the Swift Testing replacement for the `#filePath`/`#line` pair, so
  failures still point at the `trackForMemoryLeaks` call inside `makeSUT`.

> Verified on Swift 6.3.3 / Xcode 26.6: reports two issues for a deliberate `SUT ⇄ Collaborator`
> retain cycle, and passes once the back-reference is made `weak`. Confirm it fails before you
> trust it — a leak detector that has never gone red is exactly the vacuous-green trap this file
> warns about elsewhere.

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

- Test through the public API — plain `import`, never `@testable import` (the Composition Root is the one exception); a test that needs `@testable` in a feature module is a missing public seam, not a reason to keep it
- Use `makeSUT()` in every test class — never share a `sut` across tests
- Use enum-based message recording in spies — not boolean flags
- Use `trackForMemoryLeaks` in XCTest for all reference-type SUTs and collaborators
- Name test classes by behavior (`CacheFeedUseCaseTests`), not by implementation (`LocalFeedLoaderTests`)
- Do not mock infrastructure in integration tests — use the real CoreData stack

## Verification

- [ ] No `@testable import` in any feature-module test target — every assertion goes through the public API (the Composition Root's own tests may use it)
- [ ] Every test class has a `makeSUT()` factory method
- [ ] Every `makeSUT()` in XCTest calls `trackForMemoryLeaks` on reference types
- [ ] Spy collaborators use `enum ReceivedMessage` for method recording
- [ ] Store implementations conform to the appropriate `FeedStoreSpecs` protocol
- [ ] Integration tests use separate `sutToSave` and `sutToLoad` instances
- [ ] Acceptance tests inject dependencies via `SceneDelegate` convenience init
- [ ] Snapshot tests cover light mode, dark mode, and accessibility sizes
