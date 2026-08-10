# Testability as a Design Constraint

Use this when:
- A test cannot reach what it needs, and you are deciding whether to widen access or change the design
- You need to extract a protocol boundary from a test that requires real infrastructure
- You need a contract every implementation of a boundary must satisfy
- You are validating the whole composition end to end

Skip this file if:
- You need test doubles, naming, DSLs, assertions, async determinism, Swift Testing syntax or
  XCTest migration. **Use the `swift-testing-expert` skill** — it owns all of that.
- You need layer boundaries. Use `architecture-layers.md`.
- You need the wiring being tested. Use `composition-root.md`.

> **Scope.** This file covers only the places where a testing decision *is* a design decision. Each
> section below exists because trying to test something told you to change the structure — not
> because it is good test craft. Everything else about testing lives in `swift-testing-expert`,
> which is the authority; keeping a second copy here is how the two drift apart.

Jump to:
- [Test Through the Public API](#0-test-through-the-public-api-no-testable-import)
- [Extracting a Boundary](#1-extracting-a-boundary-spy-first-protocol-last)
- [Specification Pattern](#2-specification-pattern-shared-store-specs)
- [Acceptance Tests](#3-acceptance-tests)

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

## 1. Extracting a Boundary (Spy-First, Protocol-Last)

When a test would require real infrastructure (network, database, a framework singleton), do not reach for the singleton or a subclass mock:

1. Define a minimal boundary with exactly what the SUT needs — shaped by the use case, never by the infrastructure's API surface.
2. Inject it via the initializer.
3. The abstraction lives next to the consumer module; the concrete infrastructure type conforms later, in its own module.

The test drives the extraction — you discover the boundary by trying to write the test double, not by designing an interface up front. Worked through:

```swift
// Step 0 — the smell: the view model constructs and calls concrete infrastructure.
@MainActor
final class FeedViewModel {
    private let loader = RemoteFeedLoader(client: URLSessionHTTPClient(), url: feedURL)
    private(set) var feed = [FeedImage]()

    func load() async {
        feed = (try? await loader.load()) ?? []
    }
}

// Step 1 — write the test you want. It fails to compile: there is no seam to
// substitute the loader, so the only "test" possible would hit the real network.
@Test func load_deliversFeedOnLoaderSuccess() async {
    let sut = FeedViewModel(/* no way to inject a spy */)   // ← does not compile
}

// Step 2 — introduce the boundary the failing test demands. It is a single method
// with no interception need, so per this skill's own rule it is a CLOSURE, not a
// `protocol FeedLoading` (design-principles.md, Dependency Rejection):
@MainActor
final class FeedViewModel {
    private let loadFeed: () async throws -> [FeedImage]
    private(set) var feed = [FeedImage]()

    init(loadFeed: @escaping () async throws -> [FeedImage]) {
        self.loadFeed = loadFeed
    }

    func load() async {
        feed = (try? await loadFeed()) ?? []
    }
}

// The test now writes itself — a closure stub needs no conformance boilerplate:
@Test func load_deliversFeedOnLoaderSuccess() async {
    let expected = [makeImage()]
    let sut = FeedViewModel(loadFeed: { expected })

    await sut.load()

    #expect(sut.feed == expected)
}

// Step 3 — the Composition Root passes the real implementation:
FeedViewModel(loadFeed: feedService.loadRemoteFeedWithLocalFallback)
```

If the boundary genuinely has multiple implementations or needs interception (composite/decorator/fallback), the same three steps produce a protocol (`FeedStore`) instead of a closure — and the spy conforms to it in the test target.

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

> **Why this is architecture, not test craft.** Shared specs are the Liskov Substitution Principle
> made executable: one set of assertions written against the *boundary* validates every
> implementation of it. The payoff is concrete — in the case study a contributor implemented an
> entire CoreData store against this existing contract across 26 TDD commits
> ([PR #14](https://github.com/essentialdevelopercom/essential-feed-case-study/pull/14)), without
> having written the specs. Segregating the `Failable*` protocols is Interface Segregation in the
> same breath: an in-memory store cannot fail, so it must not be forced to implement failure specs.

---

## 3. Acceptance Tests

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
        // UIWindow(frame:)/UIWindow() are deprecated (NOT obsoleted) on iOS 26 —
        // "Use init(windowScene:)". UIWindowScene has no public initializer, so this
        // runtime bypass is UNSANCTIONED and may break in any release — see note below.
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

> **The window seam, honestly.** `UIWindow(frame:)` and parameterless `UIWindow()` are deprecated
> on iOS 26 ("Use init(windowScene:) instead") but **not obsoleted** — they still compile with a
> warning, which is acceptable in a test target whose warnings-as-errors policy permits it.
> `UIWindowScene` has no public initializer, so the `(UIWindowScene.self as NSObject.Type).init()`
> trick above is an **unsupported runtime bypass** — it appears nowhere in Apple's documentation
> and may break in any OS release. The pragmatic test seam is a plain `UIWindow()` with the
> deprecation warning silenced locally; reach for the runtime trick only when a test genuinely
> needs a scene-attached window, and treat it as a known liability.

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

> **Why this is architecture.** An acceptance test here is the only thing that exercises the
> Composition Root itself — the injection seam, the fallback strategy, the scheduler wiring. Break
> the composition (remove the fallback composite) and these are the tests that go red. They are
> testing *the wiring*, which is why they live beside it.

---

## Everything else

| You need | Go to |
|---|---|
| Test doubles — dummy, stub, spy, fake, mock; stubbing vs spying | `swift-testing-expert` → `test-doubles-and-strategy.md` |
| `makeSUT`, naming, DSLs, message-enum spies, one-assertion-per-test | `swift-testing-expert` → `test-design.md` |
| Deterministic async tests, async spies, cancellation, test leaks | `swift-testing-expert` → `async-testing.md` |
| `Sendable` spies, `@MainActor` in tests, suite/test isolation | `swift-testing-expert` → `isolation-and-sendability.md` |
| `@Test`, `#expect`, `#require`, `confirmation`, traits | `swift-testing-expert` → `swift-testing-syntax.md` |
| XCTest → Swift Testing, `trackForMemoryLeaks`, interop modes | `swift-testing-expert` → `xctest-migration.md` |
| Choosing a level; why the UI-test target was deleted | `swift-testing-expert` → `test-doubles-and-strategy.md` |
| Snapshot and localization test rules | `swift-testing-expert` → `test-design.md` |
| TDD rhythm, inbox checklists, legacy-code extraction | `swift-testing-expert` → `tdd-process.md` |

---

## Guardrails

- Do not widen production access for a test — expose a real public seam or change the design
- Do not reach for a singleton or a subclass mock when a test needs infrastructure — extract a boundary shaped by the use case
- Do not write per-implementation store tests — write the contract once and run it against each
- Do not conform to a `Failable*` spec an implementation cannot deterministically satisfy
- Do not substitute business logic in an acceptance test — only infrastructure
- Do not use `Task.sleep` to wait in an acceptance test — poll for the state change with a timeout

## Verification

- [ ] No `@testable import` in a feature-module test target
- [ ] Every boundary protocol was extracted because a test needed it, shaped by the use case rather than the framework
- [ ] Each store implementation runs the shared specs, and only the `Failable*` specs it can honour
- [ ] `makeSUT` in the spec tests returns the protocol, not the concrete type
- [ ] Acceptance tests inject through the Composition Root's initializer, not a debug hook
- [ ] Breaking the composition on purpose turns an acceptance test red
