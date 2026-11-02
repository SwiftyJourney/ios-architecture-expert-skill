# Feature Implementation Workflow

Use this when:
- You are building a feature end-to-end from scratch
- You need a step-by-step checklist for the clean architecture methodology
- You need to know the correct order of implementation (BDD -> domain -> API -> cache -> presentation -> UI -> composition -> tests)

Skip this file if:
- You need detailed code for a specific layer. Use `architecture-layers.md`.
- You need Composition Root wiring details. Use `composition-root.md`.
- You need test doubles, DSLs or async-test technique. Use the `swift-testing-expert` skill.

Jump to:
- [TDD rhythm](#tdd-rhythm-applies-to-every-step-below)
- [Spy-first, protocol-last](#spy-first-protocol-last)
- [The inbox checklist](#the-inbox-checklist)
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
- [Extracting a type from legacy code](#extracting-a-type-from-legacy-code)

---

## Step 0 (optional): Prototype the UI

Build a UI prototype with static data first to validate UX with zero architecture investment, then throw the logic away. The core module compiling for **both macOS and iOS** is the cheapest enforcement of zero UIKit imports in the domain/API/cache/presentation layers — a second platform fails to build the moment one leaks.

## TDD rhythm (applies to every step below)

- New types are born *inside the test file* and stay there while their behavior is unstable. The spy's `receivedMessages` enum is the draft of the eventual protocol.
- The first test asserts the SUT does nothing on init (no side effects on creation).
- One observable behavior per commit; refactor only on green.
- Move the type to the production target (adding `public`/`final`) as its own refactor step, once the collaborator boundary has stabilized.
- Add protocol conformances in separate commits, when composition actually needs them.

> This section covers only the process that **shapes the design**. Test-writing craft — naming,
> doubles, DSLs, coverage, commit discipline — belongs to the testing skill.

### Spy-first, protocol-last

This is *how* every protocol boundary in this skill was discovered. Do not design the boundary up
front:

1. Write the test against a **concrete spy class** you declare in the test file.
2. Add API to the spy **progressively**, one method at a time, as each new test demands it. You
   are not tied to a protocol yet, so you are not constantly fixing conformances.
3. When the collaboration stops changing, **extract the protocol from the spy** — containing
   exactly the methods and types the client actually needed, and nothing else.

The result is a boundary shaped by the *use case*, never by the infrastructure's API surface. A
protocol written first is a guess; a protocol extracted last is a record of what was required.

Corollary: **test what must NOT happen.** Codebases routinely assert that a method was called and
never that it was *not* called when it shouldn't have been. "Does not delete the cache when the
cache is already empty" is the test that protects against calling the right method at the wrong
time — and it is the test that reveals a use case is doing too much.

### The inbox checklist

Use cases give you a checklist for business logic — the primary course and each named error course
is a case to implement. **Infrastructure has no such document**, and nobody can write it for you:
your product owner cannot answer "what should happen if two threads insert and delete
simultaneously?" Those are engineering decisions.

So write the checklist yourself before writing the first test. For a store boundary:

| Operation | Cases |
|---|---|
| retrieve | empty cache → empty; **twice on empty → same result (no side effects)**; non-empty → found; **twice on non-empty → same result**; error *if applicable* |
| insert | to empty → stores; to non-empty → **overrides**; error *if applicable* (no permission, no space) |
| delete | on empty → no error, stays empty; on non-empty → empties it; error *if applicable* |
| all | side effects run **serially** — this is a requirement, so it gets a test |

Two things make this checklist earn its keep:

- **"If applicable"** is where Interface Segregation comes from — an in-memory store can never
  fail, a file-system store can. That distinction becomes the `Failable*` spec protocols.
- **It is splittable.** Three people can take retrieve, insert and delete independently, and the
  same list ships as documentation, or as an executable spec suite for external implementers.

Expect to get some expectations wrong. Revise the checklist; leave the unit tests behind as the
regression net.

### Triangulate at hard limits

For any boundary rule (a 7-day expiry, a status-code range), write **three** tests: just inside,
exactly on, and just outside the limit. A single one-sided test hides an inverted comparison. Keep
all three afterwards — they are what stops a later refactor from silently moving the boundary.

The same shape applies to collections: test **zero, one, and many**.

## Step 1: Define BDD Specs

Write a narrative and acceptance criteria before any code — use the `requirements-engineering` skill to turn the feature idea into a narrative plus Given/When/Then scenarios. Each scenario becomes one acceptance test in Step 9.

**Backend not ready?** Start from the part you know most about: the boundary module (models and interfaces, with only incoming arrows), so UI, API and cache can proceed in parallel. Agree a JSON payload contract, with example responses, with the backend team and build the API layer and its mapper against it; add the end-to-end test once a test endpoint exists (001_5).

---

## Step 2: Create Domain Model (Feature Layer)

```swift
// Feed/Feature/FeedImage.swift
public struct FeedImage: Hashable, Sendable {
    public let id: UUID
    public let description: String?
    public let location: String?
    public let url: URL
}
```

Checklist:
- [ ] `struct` (value type)
- [ ] `Equatable`, or `Hashable` when it is a diffable/`ForEach` identity (`ImageComment` is only `Equatable`)
- [ ] `Sendable` (for async/await boundaries)
- [ ] Foundation only — no UIKit/SwiftUI imports
- [ ] Single-method boundaries are function types (`() async throws -> [FeedImage]`); a protocol only for a cohesive multi-method contract like `FeedStore` (see `design-principles.md` § Dependency Rejection)

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
// Feed/API/FeedEndpoint.swift
public enum FeedEndpoint {
    case get(after: FeedImage? = nil)
    public func url(baseURL: URL) -> URL { ... }
}

// Feed/API/FeedItemsMapper.swift — caseless enum is this skill's recommendation;
// the case study uses `public final class` with static members
public enum FeedItemsMapper {
    private struct Root: Decodable { ... }
    public enum Error: Swift.Error { case invalidData }

    public static func map(_ data: Data, from response: HTTPURLResponse) throws -> [FeedImage] { ... }
}
```

Checklist:
- [ ] Endpoint is an `enum` with associated values while only the Composition Root uses it; a struct with static factories if it is public or churns (`architecture-layers.md` § API Layer)
- [ ] Endpoint computes URL from a `baseURL` parameter
- [ ] Mapper is a caseless `enum` with `static` methods — no instance state
- [ ] Decodable types are `private` (never leaked)
- [ ] Validates HTTP status code before decoding
- [ ] Tests: happy path, invalid data, non-2xx status codes

---

## Step 5: Build Cache Layer

```swift
// Feed/Cache/FeedStore.swift (protocol)
public protocol FeedStore {
    func deleteCachedFeed() throws
    func insert(_ feed: [LocalFeedImage], timestamp: Date) throws
    func retrieve() throws -> CachedFeed?
}

// Feed/Cache/LocalFeedImage.swift (cache model)
public struct LocalFeedImage: Equatable { ... }

// Feed/Cache/LocalFeedLoader.swift (orchestrator)
public final class LocalFeedLoader {
    public init(store: FeedStore, currentDate: @escaping () -> Date)
}

// Feed/Cache/FeedCachePolicy.swift (business rule)
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

A use case is a recipe, not a class: one type (`LocalFeedLoader`) can host save, load and validate, **one extension per use case** with its own typealiases and private helpers, and one test file per use case that builds the type through `makeSUT`. Splitting it into separate types later then breaks no tests (003_008).

---

## Step 6: Build Presentation Layer

```swift
// Feed/Presentation/LoadResourcePresenter.swift
@MainActor
public final class LoadResourcePresenter<Resource, View: ResourceView> {
    public func didStartLoading()
    public func didFinishLoading(with resource: Resource)
    public func didFinishLoading(with error: Error)
}

// Feed/Presentation/FeedImagePresenter.swift
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
- [ ] Error strings use `NSLocalizedString` with `Bundle.module` (SPM targets; `Bundle(for:)` only in plain framework targets)
- [ ] No UIKit/SwiftUI imports in presenter
- [ ] Feature-specific presenters use `static func map` (pure functions)
- [ ] Tests: verify all three state transitions (loading, success, error)
- [ ] Localization tests: verify all keys exist in all supported languages

---

## Step 7: Build UI

### UIKit path:

```swift
// FeediOS/FeedImageCellController.swift
public final class FeedImageCellController: NSObject,
    UITableViewDataSource, UITableViewDelegate, UITableViewDataSourcePrefetching,
    ResourceView, ResourceLoadingView, ResourceErrorView { ... }
```

### SwiftUI path:

```swift
// FeediOS/FeedImageRow.swift
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
// App/FeedUIComposer.swift
public static func feedComposedWith(
    feedLoader: @MainActor @escaping () async throws -> Paginated<FeedImage>,
    imageLoader: @MainActor @escaping (URL) async throws -> Data,
    selection: @MainActor @escaping (FeedImage) -> Void = { _ in }
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
func test_onLaunch_displaysRemoteFeedWhenCustomerHasConnectivity() throws {
    let store = try CoreDataFeedStore.empty
    let feed = try launch(httpClient: .online(response), store: store)
    XCTAssertEqual(feed.numberOfRenderedFeedImageViews(), 2)
}
```

Checklist:
- [ ] Test each BDD scenario from Step 1
- [ ] Use `HTTPClientStub.online/.offline` for deterministic network
- [ ] `launch` is **synchronous** — pass the real `CoreDataFeedStore.empty` (`/dev/null`, `.main` context); carry one store instance across simulated launches for cache persistence (see `testability-and-seams.md` §3)
- [ ] Inject via `SceneDelegate` convenience init
- [ ] DSL helper methods for readable assertions; `withWaitingChanges` (not `Task.sleep`) for parallel side effects

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
- [ ] Verify `Task.immediate` is used in `LoadResourcePresentationAdapter` (not bare `Task`) on iOS 26+ targets; earlier targets use plain `Task` with observable-state tests (`concurrency-at-boundaries.md`)
- [ ] Verify `Task.isCancelled` is checked after every `await` in adapter code

---

## Second feature: duplicate, then generalize

Generic abstractions (`LoadResourcePresenter`, `LoadResourcePresentationAdapter`, `RemoteLoader`) are never built speculatively for one client. The case study's rhythm, triggered by the *second* feature (comments) resembling the first (feed):

1. Duplicate the concrete component and make its tests pass.
2. Strip the feature-specific parts from the copy (e.g. the `title`).
3. Introduce the variation point with concrete placeholder types (a `Mapper` closure).
4. Make it generic (`<Resource, View: ResourceView>`).
5. Swap each feature to the generic type, one commit each.
6. Delete the now-unused concrete logic.

Never create generics for a single client — duplication across two features is the signal, and two green implementations side by side prove the abstraction's shape.

When a concrete type becomes generic, keep the old name compiling with a typealias (`typealias RemoteFeedLoader = RemoteLoader<[FeedImage]>`, plus a convenience init if needed) so no client breaks mid-refactor — "if you define a type alias, you don't break clients" (007/001). Delete the use-case tests the generic suite now duplicates, and retire the alias once the composition has moved.

> Migrating an *existing* protocol boundary (sync→async, completion→`throws`)? See the migration playbook in `concurrency-at-boundaries.md` — same incremental, tests-green-throughout discipline.

---

## Extracting a type from legacy code

**Legacy code is code with no tests.** Extracting a type out of it — into its own module, or just
out of a bloated one — has an algorithm. Follow it and it is mechanical; improvise and you will
lose behavior silently.

1. **Create a NEW type. Do not move the old one.** This is the step people skip and the one the
   whole method rests on. The new type starts empty and grows one tested behavior at a time.
2. **Open the old implementation in a split pane. It is your checklist.** Every method and branch
   in it is a behavior you owe a test.
3. **Write the failing test first, then paste.** The rule is absolute: *you only paste code once
   you have seen a failing test.* Add an empty method body to make it compile, watch it fail for
   the right reason, then bring the implementation over. Do this even for a trivial static
   property — return an empty string first and see the test go red.
4. **Follow the compiler.** After pasting, bring across only what it asks for: the protocol, then
   the view model, then the spy conformance. Bring things *partially* if only part is used.
5. **Do not test protocols.** They have no implementation. Copy them.
6. **Move resources with the code.** Localized `.strings` tables, asset catalogs, CoreData models —
   create fresh ones in the destination module rather than dragging shared files and breaking the
   source module. Verify target membership afterwards. The tests are what catch a forgotten asset.
7. **Move to production last.** Once green, create the production folder, move the types, and let
   the compiler show you exactly which API needs to become `public`. Public only what crosses the
   module boundary; test helpers and factories stay internal.
8. **Delete the old duplicate and follow the compiler errors.** Add imports, widen access, re-point
   test helpers at the new bundle, and migrate the old localization tests too.

Two notes that keep this safe:

- **Behavior parity, not code parity.** The new implementation need not be identical — it needs the
  same behavior and the same interface.
- **A type that becomes `public` must be tested in isolation.** While it was internal it could be
  covered through integration with its client; once it is part of the module's contract, a failure
  in it must point at *it*, not at whatever composed it.

Moving code between modules is delicate. Commit often, so every step has a green state to revert to.

## Guardrails

- Do not skip BDD specs — they drive the acceptance tests in Step 9
- Do not jump to UI before domain and presentation are tested
- Do not wire dependencies outside the Composition Root
- Do not add concurrency annotations retroactively — Step 3 ensures they are baked in from the start
- Do not generalize with a single client — duplicate first; extract generics only once two features share the behavior
- Do not design a protocol boundary up front — grow it on a spy and extract it last
- Do not start infrastructure work without an inbox checklist — nobody else can write it for you
- Do not move a type out of legacy code — create a new one and paste only behind failing tests

## Verification

- [ ] Every step has a passing test suite before moving to the next
- [ ] The feature decision tree in `SKILL.md` matches the workflow order
- [ ] All BDD scenarios from Step 1 are covered by acceptance tests in Step 9
