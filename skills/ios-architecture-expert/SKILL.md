---
name: ios-architecture-expert
description: >-
  Use when the user is designing, structuring, refactoring, or reviewing the
  architecture of an iOS/Swift app: separating features into modules/layers,
  choosing between MVC/MVVM/MVP, wiring dependencies in a composition root,
  breaking up an overloaded view controller, making code testable via protocol
  boundaries and test doubles, adapter/proxy/composite/decorator patterns, or
  handling Sendable/@MainActor at module boundaries. Also use it to name what
  is wrong with a design -- command-query separation, interface segregation,
  dependency inversion, coupling bottlenecks. Trigger even when 'architecture' isn't
  mentioned -- e.g. 'where should networking code live', 'my view controller
  is untestable', 'how do I split this into Swift packages', 'MVVM vs MVP',
  'should this be a protocol'. Do NOT use for pure SwiftUI view/state/styling
  work (swiftui-expert), writing BDD specs or user stories
  (requirements-engineering), Swift Testing or XCTest syntax questions, or
  non-iOS architecture (Android, backend services).
---

# iOS Architecture Expert — Clean Modular Architecture

The patterns in this skill come from the Essential Developer *Essential Feed* case study. Treat it
as a **worked example**, not as the user's codebase: the user's app has its own types, module names,
deployment target and constraints, and the job is to apply the *reasoning* to those.

## How to Apply This Skill

These come before the architecture rules, because getting them wrong makes correct architecture
useless to the person asking.

1. **Map patterns onto the user's code; don't import the case study's.** If the user showed code,
   work with their type names and signatures. If they didn't, say briefly what you're assuming and
   keep names generic to their domain. Do not "fix" types the user never mentioned
   (`LoadResourcePresenter`, `FeedService`, `ListViewController`, `Scheduler`) or assume they exist —
   introduce a pattern only when the problem calls for it, and explain it as new.
2. **Scale the deliverable to the question.** A review or "is this wrong?" question gets a
   diagnosis plus the smallest code that shows the fix. A focused refactor touches the files in
   play. Only a greenfield "build the feature" request warrants a full module layout. Don't invent
   features, modules, or test suites nobody asked for — a 4-module package is not the answer to
   "show me the refactor for the API client".
3. **Respect what the user specified.** Keep their deployment target, their requested API shape
   (`async throws` stays `async throws`), their test framework and their deadline. If you think a
   constraint is wrong, deliver what they asked for and state the trade-off in one or two sentences
   — don't silently override it. iOS 26-only APIs (`Task.immediate`, `isolated deinit`) are opt-in:
   use them only when the stated target allows, otherwise use the documented fallback; never raise a
   target to reach them.
4. **Under a deadline, fix correctly and small, then list the rest.** "Quick fix, we ship Friday"
   means: make the change that is *actually* safe (e.g. replace `@unchecked Sendable` with real
   isolation — not a silencing annotation), keep APIs stable, and give a short prioritized list of
   the architectural follow-ups (singleton removal, policy extraction, navigation, UI types out of
   the view model). Correctness is not negotiable; scope is.

## Agent Behavior Contract

When designing or reviewing architecture, follow these rules:

1. **Feature, API, cache and presentation code imports no UI framework.** Domain models, use cases, presenters and API/cache logic never depend on UIKit/SwiftUI. In an SPM split this means Foundation-only targets; the case study's core framework is *UI-free* (it also holds CoreData infrastructure in folders), which is the minimum bar.
2. **Define a boundary at every layer transition — protocol *or* function type.** A layer never references a concrete type from another layer. A **single-method** boundary defaults to a function type (`() async throws -> Resource`) composed in the Composition Root, *even with several implementations* — closures compose fallback, decoration and interception with plain code. Use a protocol when the boundary has several cohesive methods (`FeedStore`), when a named contract matters for implementers (shared contract specs, external clients), or when type-level conformance carries meaning (`ResourceView` with its associated type).
3. **Domain models are value types** — `struct`, `Equatable` (or `Hashable`), and explicitly `Sendable` when they cross concurrency boundaries (public structs are never implicitly `Sendable`). Data models carry no behavior; business models and policies (`FeedCachePolicy`) carry app-agnostic rules but no side effects.
4. **Presentation logic is framework-agnostic** — presenters output view models (structs). No `UIImage`, `UIColor`, or SwiftUI types in the presentation layer; images cross as `Data` or through an injected `(Data) -> Image?` transformer.
5. **All dependency wiring happens in the Composition Root** — the app target (or a dedicated `CompositionRoot` module). Feature modules never create their own dependencies or decide object lifetimes.
6. **Tests drive the design** — one test suite per use case, named by behavior (`CacheFeedUseCaseTests`, not `LocalFeedLoaderTests`). Use a `makeSUT()` factory in every suite so the production type can be split later without breaking tests.
7. **Test only the public API — `@testable import` only in Composition Root / app-target suites.** Feature test targets use a plain `import Feed`; a behavior that can't be observed publicly is a missing public seam (a boundary, a return value, an injected collaborator), never a reason to widen the test's access. This is deliberately stricter than the case study, which also uses `@testable` in three snapshot tests to reach an internal `ResourceErrorViewModel` initializer — see `testability-and-seams.md`.
8. **Test convenience never shapes production API.** No default parameters added so old tests compile, no conformances added only for assertions, no swappable `static var shared`, no protocols mirroring framework types just to mock them, no call-recording properties in production types.
9. **Separate modules by layer and platform** — at minimum platform-agnostic core / UI / app (composition). Start with folders in one target; split into SPM targets when boundaries need enforcing — see `spm-project-structure.md`.
10. **Prefer async/await over closures/Combine** for new async code. `Task.immediate` starts work in the same turn (what makes adapters and acceptance tests deterministic) but is iOS 26+ with no back-deployment; on earlier targets use plain `Task` and make tests await observable state — see `concurrency-at-boundaries.md`.
11. **Match the project's existing test framework; default to Swift Testing for greenfield suites.** For test doubles, test design, deterministic async testing, Swift Testing syntax and XCTest migration, defer to the **`swift-testing-expert`** skill. This skill keeps only the places where a testing decision is a *design* decision. Two things that bite regardless: `trackForMemoryLeaks`/`addTeardownBlock` is XCTest-only (the Swift Testing equivalent is a test-scoping trait, ST-0007), and Swift Testing runs tests **in parallel by default**, so suites sharing a store URL or on-disk artifacts need `.serialized`.
12. **Presenters, adapters, view controllers and composition code run on `@MainActor`.** This is a deliberate Swift 6 trade-off — the course treats "main thread" as a UIKit detail that shouldn't leak into presentation; compile-checked isolation is worth it, but a presenter still never *dispatches*. Domain types and store/client protocols stay non-isolated.
13. **Don't add a protocol nobody needs.** A protocol with one conformer and no consumer (or a single-method protocol that a function type would serve) is ceremony — compose a closure instead. The case study itself carries a few such leftovers (`FeedCache`, `FeedImageDataLoader`); don't copy them as exemplars.

---

## Architecture Diagnostic Table

| Symptom | First check | Smallest safe fix | Deep dive |
|---|---|---|---|
| Feature module imports UIKit/SwiftUI | Dependency graph | Move type to correct layer | `references/architecture-layers.md` |
| "MVVM or MVP or clean architecture?" | What problem you actually have | Start MVC; extract when duplication or cross-platform reuse appears | `references/presentation-patterns.md` |
| Presentation shared by a UIKit and a SwiftUI/watchOS client | Which clients are imperative | Foundation-only mapper/presenter shared; `@Observable` view model per SwiftUI client; MVP view protocols only if every client is imperative | `references/presentation-patterns.md` |
| View controller coordinates too much | Count its views/models/state, not its lines | Split into tiny MVCs; move creation to a composer | `references/presentation-patterns.md` |
| Presentation logic duplicated across controllers | Value transformation in the controller | Extract a platform-agnostic view model or presenter | `references/presentation-patterns.md` |
| Protocol with exactly one implementation, or a single-method protocol | Several cohesive methods? Named contract? | Replace with a function type composed in the root | `references/design-principles.md` |
| "Should this be a singleton?" | Which of the three kinds it actually is | Inject it; let the root decide the lifetime | `references/dependency-management.md` |
| Every feature depends on one shared API client | Which of the four levels you're on | Invert: each module declares the interface it needs | `references/dependency-management.md` |
| Checking connectivity before a request | There is no reliable check | Attempt the request; configure `waitsForConnectivity` | `references/infrastructure-and-networking.md` |
| `@MainActor` type races despite Swift 6 | Approachable Concurrency on + reached via a **sync** protocol requirement? | Hop executors in the root via a `Scheduler` | `references/concurrency-at-boundaries.md` |
| `#ConformanceIsolation` error on a `@MainActor` type conforming to a protocol | Approachable Concurrency off (plain Swift 6) | Isolated conformance kept out of `Sendable` contexts, `nonisolated` witnesses, or a scheduler hop | `references/concurrency-at-boundaries.md` |
| Storyboard "forces" property injection | iOS version | `instantiateViewController(identifier:creator:)` | `references/dependency-management.md` |
| Use case both returns data and mutates state | Command–Query Separation | Split into a query and a command use case | `references/design-principles.md` |
| Conformance with empty method bodies | Interface Segregation | Split the protocol, or compose optional capabilities in a struct | `references/design-principles.md` |
| Domain model has `Codable`/`@Model`/`CodingKeys` | Framework detail in an agnostic type | Add a private mirror type in the infrastructure | `references/architecture-layers.md` |
| Cross-cutting concern duplicated across UI types | Where the concrete types are known | Decorate at the composition site | `references/adapters-and-proxies.md` |
| Retain cycle between presenter and view | Which reference closes the cycle back to the presenter? | Proxy *that* reference weakly; anything owned only by the presenter stays strong | `references/adapters-and-proxies.md` |
| CoreData crashes on launch | Fallback strategy | `assertionFailure` + logged fault + in-memory (or Null Object) fallback — a product decision | `references/composition-root.md` |
| Duplicate network requests on refresh | `isLoading` guard | Use `LoadResourcePresentationAdapter` | `references/adapters-and-proxies.md` |
| Test requires real infrastructure | Protocol boundary | Extract protocol at boundary | `references/testability-and-seams.md` |
| Sendable warning at composition boundary | Closure annotations | Add `@Sendable` + `@MainActor` | `references/concurrency-at-boundaries.md` |
| Pagination won't load next page | `loadMore` closure | Verify recursive composition in the view adapter; last page has `loadMore == nil` | `references/adapters-and-proxies.md` |
| UI appends/dedups pages, or cache gets page-aware APIs | Page contract | Each page carries all items so far; the page builder merges; cache stays unaware | `references/adapters-and-proxies.md` |
| Generic list grows `hasPagination`/`showsRefresh` flags | Open/Closed | Inject behavior as content (load-more is another cell controller); the creator configures | `references/adapters-and-proxies.md` |
| Refresh error empties the list | Error state model | Keep rendered content; error is an extra state; stop loading on both paths | `references/presentation-patterns.md` |
| Cache validation doesn't run | Who triggers it | Fire-and-forget from the root on resign-active (`Task.immediate` on iOS 26+) | `references/concurrency-at-boundaries.md` |
| Debug/test-only code in the production composition | `#if DEBUG` isolation | Keep it out of the release build; prefer in-process acceptance tests over launch-argument back doors | `references/composition-root.md` |
| SwiftUI app has no obvious place to wire dependencies | Views constructing their own dependencies | Compose in `@main App`/Scene; required collaborators via init (missing = compile error), only cross-cutting values via environment (missing = runtime crash or silent default) — say which and why | `references/swiftui-composition.md` |
| SwiftUI list refetches every time the user navigates back | `.task` calling `load()` on a long-lived view model | `.task` calls a first-load guard (`loadIfNeeded`); only `.refreshable`/retry call `load()` | `references/swiftui-composition.md` |

---

## Gotchas

- `Task.immediate` (SE-0472, Swift 6.2) has runtime availability aligned to iOS 26/macOS 26 with **no back-deployment**. What it buys is starting the loader in the same turn (the request is registered synchronously, so call-count assertions and acceptance-test launches are deterministic). `didStartLoading()` is called *before* the Task in the case study, so it's synchronous either way. On earlier targets use plain `Task` and make tests await observable state.
- A CoreData store at `URL(fileURLWithPath: "/dev/null")` with a `.main` context queue is effectively in-memory AND synchronous — it's what makes acceptance tests deterministic without sleeps.
- In a `@MainActor` service, closures handed to a non-isolated scheduler capture the *values* (`[store, logger]`), not main-actor state through `self`. Capturing `self` is legal (a `@MainActor` class is `Sendable`); reading `self.store` (a main-actor `lazy var`) from the non-isolated closure is the compile error.
- **Public structs are never implicitly `Sendable`** (only internal/fileprivate ones get inference). Write `Sendable` on public value types that cross boundaries. Global-actor-isolated classes, by contrast, *are* implicitly `Sendable`.
- `UIWindow(frame:)` and `UIWindow()` are deprecated on iOS 26 ("Use init(windowScene:)") — deprecated, **not obsoleted**: they still compile with a warning, which is acceptable in test targets. The `(UIWindowScene.self as NSObject.Type).init() as? UIWindowScene` dummy-scene trick works (no public initializer exists) but is an **unsanctioned runtime bypass** that may break in any release — see `testability-and-seams.md`.
- `didEndDisplaying(cell:forRowAt:)` fires for OLD cells after a data-source update — never cancel by indexing a plain array model. With a diffable data source, look the controller up with `dataSource.itemIdentifier(for: indexPath)`; it owns the old/new mapping (the case study removed its hand-rolled `[IndexPath: CellController]` registry when it adopted diffable).
- Diffable data sources don't reflow on Dynamic Type changes — observe `registerForTraitChanges([UITraitPreferredContentSizeCategory.self]) { ... }` (`traitCollectionDidChange(_:)` is deprecated since iOS 17) and call `tableView.reloadData()`. Apply data updates with `dataSource.apply(snapshot)` so only changed rows update.
- A 2xx response with an **empty body** (e.g. 204) is a valid success — the URL Loading System delivers zero-byte `Data`. The HTTP client must not reject it; whether empty is acceptable is the per-endpoint mapper's decision.
- Day-based business rules (cache max age) use `Calendar.date(byAdding: .day, ...)` with an explicit calendar, never `n * 86_400` seconds — not every day has 24 hours. A `nil` date means "invalid", not a crash.
- `Thread.isMainThread` is not "on the main queue" (the main thread can run other queues) — prefer `@MainActor` isolation over runtime checks.
- Load `NSManagedObjectModel` once and cache it statically — loading it twice registers duplicate `NSEntityDescription`s claiming the same `NSManagedObject` subclasses (undefined behavior).
- A diffable data source has **no sections at all** until the first snapshot is applied — guard `numberOfSections` in test DSLs, or they crash before the first render.
- `adjustsFontForContentSizeCategory` defaults to **`false`** — `UIFont.preferredFont(forTextStyle:)` alone does not give you Dynamic Type.
- An `NSKeyValueObservation` must be **retained** or the observation is removed immediately; release it deliberately (e.g. in `didEndDisplaying`).
- Diffable data sources request cells **eagerly**, ahead of display, based on `estimatedRowHeight` — set a realistic estimate, and move expensive work from `cellForRowAt` to `willDisplay`.
- **With Approachable Concurrency on, a `@MainActor` type reached through a non-isolated *synchronous* protocol requirement can run off the main actor silently.** The compiler infers an *isolated conformance* (SE-0470); called through an existential from a non-isolated context, nothing warns or traps. In plain Swift 6 (setting off) the same conformance is a `#ConformanceIsolation` compile error instead. The case study enables Approachable Concurrency on every target and hops executors in the Composition Root via `Scheduler`; only its UIKit module uses default `MainActor` isolation (hence `nonisolated Equatable`/`Hashable` conformances there).
- `URLSession` holds a **strong** reference to its delegate until you invalidate the session — `invalidateAndCancel()` or you leak until the app exits.
- Setting anything on a `URLSessionConfiguration` **after** creating the session has no effect.
- `Task` is not `AnyCancellable`: it does not cancel on `deinit`, and `Task.isCancelled` must be checked manually. When migrating from Combine, the `Deferred { Future { } }` + `sink` pipeline started work synchronously at subscription; plain `Task` enqueues it, and `Task.immediate` restores the synchronous start.

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
| `WeakRefVirtualProxy<T>` | Break the one reference that closes a presenter->view cycle, via conditional conformance |
| `LoadResourcePresentationAdapter` | Generic async loader bridging use cases to presenters with cancellation |
| `FeedViewAdapter` | Maps domain models to `CellController` array, composes recursive `loadMore` |
| `Paginated<Item>` | Each page carries all items so far + optional `loadMore` (`nil` on the last page); UI is keyset/offset-agnostic |
| Composition Root / `FeedService` | `@MainActor` orchestrator with lazy init, Scheduler, and fallback strategy |
| `Scheduler` protocol | Abstract store execution context; mandatory for CoreData (sync store calls are only valid on the context queue) |
| `InMemoryFeedStore` | `@MainActor`, NSCache-backed PRODUCTION fallback when CoreData fails to init — not a test double (acceptance tests use the real `CoreDataFeedStore` at `/dev/null`) |
| `LoaderSpy<Param, Resource>` | Generic async test spy using `AsyncThrowingStream` for UI integration tests |
| Specification Pattern | Protocol-driven shared test specs across store implementations |
| UI Composer (static factory) | Wire presenter->adapter->view chain per feature (`FeedUIComposer.feedComposedWith`) |
| Static Mapper | Pure function for data transformation with per-endpoint status rules — `FeedItemsMapper.map(_:from:)` |
| Cache Policy | Business rule encapsulation for cache validation — `FeedCachePolicy.validate(_:against:)` |
| `CellController` | Type-erased cell composition — wraps `UITableViewDataSource` + `Delegate` + `Prefetching` |

---

## Feature Decision Tree

Starting a new feature? Follow this path:

0. **Backend not ready?** -> Start from the boundary module (models + interfaces, only incoming arrows) and agree a JSON payload contract with the backend team
1. **Define the domain model** -> `struct` in Feature layer, `Equatable`/`Hashable`, explicit `Sendable`
2. **Add concurrency annotations** -> `@MainActor` on view protocols, `Sendable` on models
3. **Need remote data?** -> Endpoint + static mapper in API layer (mapper owns the status-code rule)
4. **Need persistence?** -> Do you need it at all (`URLCache` may do)? Otherwise store protocol + local model + cache policy in Cache layer
5. **Need to display it?** -> Start with MVC and split into tiny MVCs; reach for a view model or presenter only when duplication or cross-platform reuse demands it. Choose by client: SwiftUI clients get `@Observable` view models over a shared Foundation-only presenter/mapper. Once two features share the loading/error/success shape, generalize to a `LoadResourcePresenter`-style generic
6. **UIKit or SwiftUI?** -> Build the view layer; UIKit views conform to the presenter's view protocols
7. **Wire it up** -> Composer + adapter (+ proxy where a cycle exists) in Composition Root
8. **Verify concurrency** -> Build with `SWIFT_STRICT_CONCURRENCY = complete`, run Thread Sanitizer

> Step-by-step guide: [feature-implementation-workflow.md](references/feature-implementation-workflow.md)

---

## Guardrails

- Do not create concrete types inside feature modules — all instantiation belongs in the Composition Root
- Do not reach for `.shared` from a component — inject it and let the Composition Root own the lifetime (`lazy var` in `FeedService` gives a shared instance without global state). Note "singleton" means three different things — see `dependency-management.md`
- Do not reference a DI container, coordinator or router from a feature module — that is the Service Locator anti-pattern
- Do not check connectivity before making a request
- Do not add `@MainActor` to domain types or store/client protocols — isolation belongs to presentation, adapters, composition, and specific infrastructure implementations (e.g. an in-memory store), never to the boundary
- Do not use `@unchecked Sendable` — redesign the type as a value type or use actor isolation
- Do not protect state reached from async code with a lock — use `@MainActor` (light work) or an actor; `Mutex` is for short critical sections in synchronous code only
- Do not embed cache policy logic inside the loader — keep it as a separate type
- Do not navigate from a view controller when the destination needs a non-trivial object graph or the controller is generic/reusable — emit a selection closure the Composition Root wires. A simple push or alert with data already in hand may stay ("when in doubt, go simple"). The container (nav/tab/modal) is always the composition's choice
- Do not use `@testable import` outside Composition Root / app-target suites; widen production access (`public`) intentionally rather than tunneling into `internal`/`private` from tests
- Do not wipe rendered content on a refresh error — show the error as an extra state and stop the loading indicator on both paths
- Do not add per-feature flags to a generic component (`hasPagination`, `showsRefresh`) — inject the behavior as content and let the creator configure it
- Do not generalize with a single client. The case study's rhythm is duplicate → generalize → replace → delete: copy the working concrete component for the second feature, and only extract generics (`LoadResourcePresenter`-style) once two green implementations exist side by side
- Do not encode memory-management policy in a component — lifetime is the composer's decision, which is why the presenter holds its view strongly and the composer proxies only the reference that closes a cycle
- Do not let a feature module depend on the Composition Root — invert it and pass a closure in
- Do not let feature modules import shared generic modules as the end state — features expose mappers and platform conformances; the root joins them to shared generics. Coupling two sibling features is a judgment call (worth decoupling mainly when separate teams own them)
- Do not reach for coordinators, routers or a DI container before the complexity justifies them; when you do, they live in the Composition Root only
- Do not make a domain or store boundary `async` because one implementation is — keep it synchronous and inject the execution context at composition time
- Defer non-architectural concurrency questions (task groups, async sequences, actor reentrancy) to the `swift-language-expert` skill

---

## Verification Checklist

When implementing or reviewing architecture:

1. Build with `SWIFT_STRICT_CONCURRENCY = complete` — zero warnings
2. No UIKit/SwiftUI imports in Feature/API/Cache/Presentation modules — deleting those imports from every presentation file still compiles
3. All protocol boundaries have corresponding test doubles
4. Feature test targets `import` modules plainly — `@testable` appears only in Composition Root / app-target suites; every other assertion goes through the public API
5. `makeSUT()` exists in every test suite
6. Store initialization failure is handled deliberately (in-memory fallback or Null Object — a logged product decision)
7. Run Thread Sanitizer (`-enableThreadSanitizer YES`) — zero data races
8. In UIKit composition, the reference that closes each presenter↔view cycle is weak (`WeakRefVirtualProxy`) and everything owned only by the presenter is strong; in SwiftUI, ownership is via `@State`/`@Observable`
9. Pagination `loadMore` is `nil` for the last page
10. Every protocol earns its place: several cohesive methods, a named contract, or meaningful type-level conformance — otherwise a function type
11. No conformance in the codebase has an empty method body
12. No feature module imports the Composition Root
13. Nothing in production exists only to make a test compile or assert (default params, conformances, swappable globals, recording properties)

---

## Reference Router

Open the smallest reference that matches the question:

- **Why, before how**
  - [design-principles.md](references/design-principles.md) — symptom→principle table, CQS, ISP, dependency inversion & rejection, DTOs, invalid states, Functional Core/Imperative Shell
- **Dependencies & Ownership**
  - [dependency-management.md](references/dependency-management.md) — singleton taxonomy, the four levels, injection techniques, composer rules, Pure DI vs containers, lifetimes
- **Infrastructure**
  - [infrastructure-and-networking.md](references/infrastructure-and-networking.md) — connectivity (never pre-flight), URLSession configuration & delegate lifetime, representable states, CoreData details, logging & fallbacks
- **Architecture & Layers**
  - [architecture-layers.md](references/architecture-layers.md) — layer boundaries, domain models, protocols, do-you-need-persistence
  - [spm-project-structure.md](references/spm-project-structure.md) — onion, horizontal/vertical slicing, module layout, Package.swift, CI
- **Presentation**
  - [presentation-patterns.md](references/presentation-patterns.md) — MVC/MVVM/MVP, diagnosing a bloated controller, where memory policy lives
- **SwiftUI**
  - [swiftui-composition.md](references/swiftui-composition.md) — `@main App` composition root, environment DI, NavigationStack routing at the root, @Observable presenters, acceptance seams
- **Composition & Wiring**
  - [composition-root.md](references/composition-root.md) — what the root is, lifetimes, FeedService, Scheduler, fallback, logging
  - [adapters-and-proxies.md](references/adapters-and-proxies.md) — adapter, proxy, composite, decorator, interception, pagination wiring
- **Concurrency in Architecture**
  - [concurrency-at-boundaries.md](references/concurrency-at-boundaries.md) — why boundaries stay synchronous, Scheduler, @Sendable, Task.immediate, cancellation
- **Testing**
  - [testability-and-seams.md](references/testability-and-seams.md) — testing decisions that are design decisions: public-API testing, extracting boundaries, contract specs, acceptance tests through the root
  - Everything else about testing → the **`swift-testing-expert`** skill
- **Workflow**
  - [feature-implementation-workflow.md](references/feature-implementation-workflow.md) — step-by-step feature building, spy-first/protocol-last, legacy-code extraction
