---
name: ios-architecture-expert
description: >-
  Use when the user is designing, structuring, refactoring, or reviewing the
  architecture of an iOS/Swift app: separating features into modules/layers
  (domain, API, cache, presentation, UI), choosing between MVC/MVVM/MVP,
  wiring dependencies in a composition root, breaking up a view controller
  that coordinates too much, making code testable via protocol boundaries and
  test doubles, adapter/proxy/composite/decorator patterns, or handling
  Sendable/@MainActor at module boundaries. Also use it to name what is wrong
  with a design -- command-query separation, interface segregation, dependency
  inversion, coupling bottlenecks. Trigger even when 'architecture' isn't
  mentioned -- e.g. 'where should networking code live', 'my view controller
  is untestable', 'how do I split this into Swift packages', 'MVVM vs MVP',
  'should this be a protocol'. Do NOT use for pure SwiftUI view/state/styling
  work (swiftui-expert), writing BDD specs or user stories
  (requirements-engineering), Swift Testing or XCTest syntax questions, or
  non-iOS architecture (Android, backend services).
---

# iOS Architecture Expert — Clean Modular Architecture

## Agent Behavior Contract

When this skill is active, follow these rules **strictly**:

1. **Feature modules have zero UIKit/SwiftUI imports** — only Foundation. Domain models, use cases, presenters, and API/cache logic never depend on a UI framework.
2. **Define a boundary at every layer transition — protocol *or* closure.** A layer never references a concrete type from another layer. Use a protocol when there are genuinely multiple strategies (`FeedStore`, `HTTPClient`, `ResourceView`, `FeedCache`); use a plain composed closure (`() async throws -> Resource`) when there is one — see rule 11.
3. **Domain models are value types** — `struct`, `Hashable`, `Sendable`. No classes for models.
4. **Presentation logic is framework-agnostic** — presenters output view models (structs). No `UIImage`, `UIColor`, or SwiftUI types in the presentation layer.
5. **All dependency wiring happens in the Composition Root** — the app target (or a dedicated `CompositionRoot` module). Feature modules never create their own dependencies.
6. **Tests drive the design** — one test class per use case, named by behavior (`CacheFeedUseCaseTests`, not `LocalFeedLoaderTests`). Use `makeSUT()` factory in every test class.
6b. **Test only the public API — no `@testable import`.** Test targets use a plain `import Feed`, never `@testable import Feed`. Tests exercise a module exclusively through its public surface; they never reach for `internal`/`private` members. If a behavior can't be observed publicly, that's a design signal — expose it through the module's public API (a protocol boundary, a return value, an injected collaborator/spy), not by widening access for the test. Same-target tests (e.g. acceptance tests in the app target) still assert through the composed public seams, not private state. **One exception:** the Composition Root has no external clients, so its types never need to be `public` and `@testable import App` is correct there — and only there.
7. **Use SPM multi-target packages** for module separation — `Feed` (Foundation), `FeediOS` (UIKit), `App` (composition).
8. **Prefer async/await over closures/Combine** for async operations. Use `Task.immediate` for synchronous-first execution in adapters.
9. **Match the project's existing test framework; default to Swift Testing for greenfield suites.** The reference patterns in `testing-strategy.md` are XCTest, as in the source codebase. Translating is mostly mechanical — the one real gap is `trackForMemoryLeaks`/`addTeardownBlock`, which is XCTest-only; the Swift Testing equivalent is a test-scoping trait (ST-0007, Swift 6.1). Also note Swift Testing runs tests **in parallel by default**, so suites sharing a store URL or on-disk artifacts need `.serialized`.
10. **Mark shared mutable state with `@MainActor`** — presenters, adapters, view controllers, and composition code run on the main actor.
11. **Do not add a protocol boundary that has only one implementation** and no second strategy in sight — compose a closure in the Composition Root instead. Boundaries exist to select between strategies.

---

## Architecture Diagnostic Table

| Symptom | First check | Smallest safe fix | Deep dive |
|---|---|---|---|
| Feature module imports UIKit/SwiftUI | Dependency graph | Move type to correct layer | `references/architecture-layers.md` |
| "MVVM or MVP or clean architecture?" | What problem you actually have | Start MVC; extract when duplication or cross-platform reuse appears | `references/presentation-patterns.md` |
| View controller coordinates too much | Count its views/models/state, not its lines | Split into tiny MVCs; move creation to a composer | `references/presentation-patterns.md` |
| Presentation logic duplicated across controllers | Value transformation in the controller | Extract a platform-agnostic view model or presenter | `references/presentation-patterns.md` |
| Protocol with exactly one implementation | Is there a second strategy? | Delete it; compose a closure in the root | `references/design-principles.md` |
| Use case both returns data and mutates state | Command–Query Separation | Split into a query and a command use case | `references/design-principles.md` |
| Conformance with empty method bodies | Interface Segregation | Split the protocol, or compose optional capabilities in a struct | `references/design-principles.md` |
| Domain model has `Codable`/`@Model`/`CodingKeys` | Framework detail in an agnostic type | Add a private mirror type in the infrastructure | `references/architecture-layers.md` |
| Cross-cutting concern duplicated across UI types | Where the concrete types are known | Decorate at the composition site | `references/adapters-and-proxies.md` |
| Retain cycle between presenter and view | Proxy wiring | Add `WeakRefVirtualProxy` | `references/adapters-and-proxies.md` |
| CoreData crashes on launch | Fallback strategy | Add `InMemoryFeedStore` fallback | `references/composition-root.md` |
| Duplicate network requests on refresh | `isLoading` guard | Use `LoadResourcePresentationAdapter` | `references/adapters-and-proxies.md` |
| Test requires real infrastructure | Protocol boundary | Extract protocol at boundary | `references/testing-strategy.md` |
| Sendable warning at composition boundary | Closure annotations | Add `@Sendable` + `@MainActor` | `references/concurrency-at-boundaries.md` |
| Pagination won't load next page | `loadMore` closure | Verify recursive composition in `FeedViewAdapter` | `references/composition-root.md` |
| Cache validation doesn't run | `Task.immediate` usage | Use fire-and-forget with `Task.immediate` | `references/concurrency-at-boundaries.md` |
| Debug/test-only code in the production composition | `#if DEBUG` isolation | Move it to a `#if DEBUG` `SceneDelegate` subclass | `references/composition-root.md` |

---

## Gotchas

- `Task.immediate` requires a Swift 6.2-era toolchain/recent deployment target (the case study targets iOS 26). On older targets, fall back to plain `Task` and handle the synchronous `didStartLoading` requirement explicitly.
- `CoreDataFeedStore(storeURL: URL(fileURLWithPath: "/dev/null"), contextQueue: .main)` is effectively in-memory AND synchronous — it's what makes acceptance tests deterministic without sleeps.
- Capture `[store]`, never `[self]`, in `@Sendable` closures created inside the `@MainActor` service — capturing self drags MainActor isolation into the closure (compile error under Swift 6).
- `UIWindow(frame:)` is deprecated on iOS 26; tests build a dummy scene via `(UIWindowScene.self as NSObject.Type).init() as? UIWindowScene` (no public initializer exists).
- `didEndDisplaying(cell:forRowAt:)` fires for OLD cells after a data-source update — never cancel by indexing the NEW model; cancel through a `[IndexPath: CellController]` registry populated in `cellForRowAt`.
- Diffable data sources don't reflow on Dynamic Type changes — reload when `preferredContentSizeCategory` changes; use `applySnapshotUsingReloadData` guarded by an `itemIdentifiers` equality check.
- `Thread.isMainThread` is not "on the main queue" (the main thread can run other queues) — prefer `@MainActor` isolation over runtime checks.
- Load `NSManagedObjectModel` once and cache it statically — loading it twice registers duplicate `NSEntityDescription`s claiming the same `NSManagedObject` subclasses (undefined behavior).
- A diffable data source has **no sections at all** until the first snapshot is applied — guard `numberOfSections` in test DSLs, or they crash before the first render.
- `adjustsFontForContentSizeCategory` defaults to **`false`** — `UIFont.preferredFont(forTextStyle:)` alone does not give you Dynamic Type.
- An `NSKeyValueObservation` must be **retained** or the observation is removed immediately; release it deliberately (e.g. in `didEndDisplaying`).
- Diffable data sources request cells **eagerly**, ahead of display, based on `estimatedRowHeight` — set a realistic estimate, and move expensive work from `cellForRowAt` to `willDisplay`.

---

## Architecture Layers

| Layer | Responsibility |
|-------|---------------|
| **Feature** | Domain models (`struct`, `Sendable`) and abstract use-case protocols. Zero framework imports beyond Foundation. |
| **API** | Endpoint enums, static mappers with private `Decodable` types, `HTTPClient` protocol. |
| **Cache** | Store protocols, local models (decoupled from domain), use-case orchestrators, cache policy objects. |
| **Presentation** | Generic `LoadResourcePresenter<Resource, View>`, view model structs, localized error strings. |
| **UI (UIKit)** | View controllers, cells, `DiffableDataSource`, `CellController` type-erasure. Conforms to presenter view protocols. |
| **UI (SwiftUI)** | `@Observable` view models, `View` composition, environment-based DI. Same presenter patterns, different binding. |
| **Composition** | `Composer` static factories, `PresentationAdapter`, `WeakRefVirtualProxy`, `FeedViewAdapter`. App-target only. |

Which layer a line of code belongs to is decided by **what kind of logic it is**:
application-specific (use cases), application-agnostic (business models and policies), or framework
(infrastructure). That distinction — not the folder name — is the rule.

> Full code examples: [architecture-layers.md](references/architecture-layers.md)
> · The reasoning: [design-principles.md](references/design-principles.md)
> · Choosing MVC/MVVM/MVP for the presentation + UI layers: [presentation-patterns.md](references/presentation-patterns.md)

---

## Key Patterns Quick Reference

| Pattern | Purpose |
|---------|---------|
| Adapter | Connect two APIs that don't match (`LoadResourcePresentationAdapter`, `FeedViewAdapter`) |
| Decorator | Add behaviour keeping the same interface — cross-cutting concerns at the composition site |
| Composite | Combine implementations behind one interface — fallback, retry, layered caching |
| Interception | Inject a side effect at the composition site so neither collaborator knows (cache-on-success) |
| Null Object | Neutral behaviour instead of optionality — the production sibling of a dummy double |
| `LoadResourcePresenter<Resource, View>` | Reusable loading/error/success state machine with generic mapper |
| `WeakRefVirtualProxy<T>` | Break retain cycles in presenter->view binding via conditional conformance |
| `LoadResourcePresentationAdapter` | Generic async loader bridging use cases to presenters with cancellation |
| `FeedViewAdapter` | Maps domain models to `CellController` array, composes recursive `loadMore` |
| `Paginated<Item>` | Recursive pagination with optional `loadMore` closure (`Sendable`) |
| Composition Root / `FeedService` | `@MainActor` orchestrator with lazy init, Scheduler, and fallback strategy |
| `Scheduler` protocol | Abstract store execution context for CoreData/InMemory polymorphism |
| `InMemoryFeedStore` | `@MainActor`, NSCache-backed PRODUCTION fallback when CoreData fails to init — not a test double (acceptance tests use the real `CoreDataFeedStore` at `/dev/null`) |
| `LoaderSpy<Param, Resource>` | Generic async test spy using `AsyncThrowingStream` for UI integration tests |
| Specification Pattern | Protocol-driven shared test specs across store implementations |
| UI Composer (static factory) | Wire presenter->adapter->view chain per feature (`FeedUIComposer.feedComposedWith`) |
| Static Mapper | Pure function for data transformation — `FeedItemsMapper.map(_:from:)` |
| Cache Policy | Business rule encapsulation for cache validation — `FeedCachePolicy.validate(_:against:)` |
| `CellController` | Type-erased cell composition — wraps `UITableViewDataSource` + `Delegate` + `Prefetching` |

---

## Feature Decision Tree

Starting a new feature? Follow this path:

1. **Define the domain model** -> `struct` in Feature layer, `Hashable`, `Sendable`
2. **Add concurrency annotations** -> `@MainActor` on view protocols, `Sendable` on models
3. **Need remote data?** -> Endpoint enum + static mapper in API layer
4. **Need persistence?** -> Store protocol + local model + cache policy in Cache layer
5. **Need to display it?** -> Start with MVC and split into tiny MVCs; reach for a view model or presenter only when duplication or cross-platform reuse demands it. Once two features share the loading/error/success shape, generalize to `LoadResourcePresenter` + view model struct in the Presentation layer
6. **UIKit or SwiftUI?** -> Build view layer, conform to `ResourceView` protocols
7. **Wire it up** -> Composer + adapter + proxy in Composition Root
8. **Verify concurrency** -> Build with `SWIFT_STRICT_CONCURRENCY = complete`, run Thread Sanitizer

> Step-by-step guide: [feature-implementation-workflow.md](references/feature-implementation-workflow.md)

---

## Guardrails

- Do not create concrete types inside feature modules — all instantiation belongs in the Composition Root
- Do not use singletons — `lazy var` in `FeedService` achieves deferred init without global state
- Do not add `@MainActor` to domain types or store protocols — only presentation, adapters, and composition
- Do not use `@unchecked Sendable` — redesign the type as a value type or use `@MainActor`
- Do not embed cache policy logic inside the loader — keep it as a separate type
- Do not put navigation logic in view controllers — use closure callbacks wired in the Composition Root
- Do not use `@testable import` — test through the public API only; widen production access (`public`) intentionally rather than tunneling into `internal`/`private` from tests
- Do not generalize with a single client. The case study's rhythm is duplicate → generalize → replace → delete: copy the working concrete component for the second feature, and only extract generics (`LoadResourcePresenter`-style) once two green implementations exist side by side
- Do not encode memory-management policy in a component — lifetime is the composer's decision, which is why the presenter holds its view strongly and the composer wraps it in a proxy
- Do not let a feature module depend on the Composition Root — invert it and pass a closure in
- Do not reach for coordinators, routers or a DI container before the complexity justifies them; when you do, they live in the Composition Root only
- Do not make a domain or store boundary `async` because one implementation is — keep it synchronous and inject the execution context at composition time
- Defer non-architectural concurrency questions (task groups, async sequences, actor reentrancy) to the `swift-language-expert` skill

---

## Verification Checklist

When implementing or reviewing architecture:

1. Build with `SWIFT_STRICT_CONCURRENCY = complete` — zero warnings
2. No UIKit/SwiftUI imports in Feature/API/Cache modules
3. All protocol boundaries have corresponding test doubles
3b. Test targets `import` modules plainly — grep for `@testable` returns nothing; every assertion goes through the public API
4. `makeSUT()` exists in every test class
5. CoreData store has `InMemoryFeedStore` fallback in Composition Root
6. Run Thread Sanitizer (`-enableThreadSanitizer YES`) — zero data races
7. `WeakRefVirtualProxy` wraps all view references in UIKit composition (or `@Observable` in SwiftUI)
8. Pagination `loadMore` is `nil` for the last page
9. Every protocol boundary has, or is credibly about to have, more than one implementation
10. No conformance in the codebase has an empty method body
11. No feature module imports the Composition Root
12. Deleting `import UIKit`/`import SwiftUI` from every presentation file still compiles

---

## Reference Router

Open the smallest reference that matches the question:

- **Why, before how**
  - [design-principles.md](references/design-principles.md) — symptom→principle table, CQS, ISP, dependency inversion & rejection, DTOs, invalid states
- **Architecture & Layers**
  - [architecture-layers.md](references/architecture-layers.md) — layer boundaries, domain models, protocols, do-you-need-persistence
  - [spm-project-structure.md](references/spm-project-structure.md) — onion, horizontal/vertical slicing, module layout, Package.swift, CI
- **Presentation**
  - [presentation-patterns.md](references/presentation-patterns.md) — MVC/MVVM/MVP, diagnosing a bloated controller, where memory policy lives
- **Composition & Wiring**
  - [composition-root.md](references/composition-root.md) — what the root is, lifetimes, FeedService, Scheduler, fallback, logging
  - [adapters-and-proxies.md](references/adapters-and-proxies.md) — adapter, proxy, composite, decorator, interception, pagination wiring
- **Concurrency in Architecture**
  - [concurrency-at-boundaries.md](references/concurrency-at-boundaries.md) — why boundaries stay synchronous, Scheduler, @Sendable, Task.immediate, cancellation
- **Testing**
  - [testing-strategy.md](references/testing-strategy.md) — boundaries as testability seams, contract specs, integration/snapshot/acceptance layers, and the Swift Testing delta
- **Workflow**
  - [feature-implementation-workflow.md](references/feature-implementation-workflow.md) — step-by-step feature building, spy-first/protocol-last, legacy-code extraction
