---
name: ios-architecture-expert
description: >-
  Use when the user is designing, structuring, refactoring, or reviewing the
  architecture of an iOS/Swift app: separating features into modules/layers
  (domain, API, cache, presentation, UI), wiring dependencies in a composition
  root, breaking up massive view controllers, making code testable via
  protocol boundaries and test doubles, adapter/proxy/pagination patterns, or
  handling Sendable/@MainActor at module boundaries. Trigger even when
  'architecture' isn't mentioned -- e.g. 'where should networking code live',
  'my view controller is untestable', 'how do I split this into Swift
  packages', 'MVVM vs MVP'. Do NOT use for pure SwiftUI view/state/styling
  work (swiftui-expert), writing BDD specs or user stories
  (requirements-engineering), Swift Testing syntax (swift-testing-expert), or
  non-iOS architecture (Android, backend services).
---

# iOS Architecture Expert — Clean Modular Architecture

## Agent Behavior Contract

When this skill is active, follow these rules **strictly**:

1. **Feature modules have zero UIKit/SwiftUI imports** — only Foundation. Domain models, use cases, presenters, and API/cache logic never depend on a UI framework.
2. **Define protocol boundaries at every layer transition** — `FeedStore`, `HTTPClient`, `ResourceView`, `FeedCache`, etc. Concrete types live behind protocols.
3. **Domain models are value types** — `struct`, `Hashable`, `Sendable`. No classes for models.
4. **Presentation logic is framework-agnostic** — presenters output view models (structs). No `UIImage`, `UIColor`, or SwiftUI types in the presentation layer.
5. **All dependency wiring happens in the Composition Root** — the app target (or a dedicated `CompositionRoot` module). Feature modules never create their own dependencies.
6. **Tests drive the design** — one test class per use case, named by behavior (`CacheFeedUseCaseTests`, not `LocalFeedLoaderTests`). Use `makeSUT()` factory in every test class.
6b. **Test only the public API — no `@testable import`.** Test targets use a plain `import Feed`, never `@testable import Feed`. Tests exercise a module exclusively through its public surface; they never reach for `internal`/`private` members. If a behavior can't be observed publicly, that's a design signal — expose it through the module's public API (a protocol boundary, a return value, an injected collaborator/spy), not by widening access for the test. Same-target tests (e.g. acceptance tests in the app target) still assert through the composed public seams, not private state.
7. **Use SPM multi-target packages** for module separation — `Feed` (Foundation), `FeediOS` (UIKit), `App` (composition).
8. **Prefer async/await over closures/Combine** for async operations. Use `Task.immediate` for synchronous-first execution in adapters.
9. **Write new tests in the project's test framework.** The reference patterns here are XCTest, as in the source codebase. When the project uses Swift Testing, translate via the `swift-testing-expert` skill — note `trackForMemoryLeaks`/`addTeardownBlock` is XCTest-only.
10. **Mark shared mutable state with `@MainActor`** — presenters, adapters, view controllers, and composition code run on the main actor.

---

## Architecture Diagnostic Table

| Symptom | First check | Smallest safe fix | Deep dive |
|---|---|---|---|
| Feature module imports UIKit/SwiftUI | Dependency graph | Move type to correct layer | `references/architecture-layers.md` |
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

> Full code examples: [architecture-layers.md](references/architecture-layers.md)

---

## Key Patterns Quick Reference

| Pattern | Purpose |
|---------|---------|
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
5. **Need to display it?** -> `LoadResourcePresenter` + view model struct in Presentation layer
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
- Defer non-architectural concurrency questions to the `swift-concurrency` skill

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

---

## Reference Router

Open the smallest reference that matches the question:

- **Architecture & Layers**
  - [architecture-layers.md](references/architecture-layers.md) — layer boundaries, domain models, protocols
  - [spm-project-structure.md](references/spm-project-structure.md) — module layout, Package.swift, CI
- **Composition & Wiring**
  - [composition-root.md](references/composition-root.md) — FeedService, Scheduler, fallback, dependency creation
  - [adapters-and-proxies.md](references/adapters-and-proxies.md) — adapter, proxy, view adapter, pagination wiring
- **Concurrency in Architecture**
  - [concurrency-at-boundaries.md](references/concurrency-at-boundaries.md) — Scheduler, @Sendable, Task.immediate, cancellation
- **Testing**
  - [testing-strategy.md](references/testing-strategy.md) — unit, spec, integration, snapshot, acceptance
- **Workflow**
  - [feature-implementation-workflow.md](references/feature-implementation-workflow.md) — step-by-step feature building
