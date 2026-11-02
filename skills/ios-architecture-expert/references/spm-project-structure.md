# SPM Project Structure

Use this when:
- You need to set up an SPM multi-target package for a modular iOS app
- You need a `Package.swift` example with correct dependencies and resources
- You need the dependency graph between modules
- You need Swift 6 build settings for strict concurrency
- You need CI/CD configuration for GitHub Actions

Skip this file if:
- You need to understand what code goes in each layer. Use `architecture-layers.md`.
- You need Composition Root wiring. Use `composition-root.md`.

Jump to:
- [Onion architecture](#onion-architecture)
- [Horizontal vs vertical slicing](#horizontal-vs-vertical-slicing)
- [How physically separated should modules be?](#how-physically-separated-should-modules-be)
- [Module Layout](#module-layout) (+ [consuming the package](#how-the-app-consumes-the-package), [inside a module](#inside-a-module))
- [Package.swift Example](#packageswift-example)
- [Adding a second feature](#adding-a-second-feature--what-slicing-looks-like-on-disk)
- [Where the Composition Root lives](#where-the-composition-root-lives)
- [Static vs dynamic linking](#static-vs-dynamic-linking)
- [Build Settings](#build-settings--swift-6-strict-concurrency)
- [Resource Declarations](#resource-declarations)
- [Dependency Graph](#dependency-graph)
- [CI/CD](#cicd-github-actions-example)

> The case study itself ships as **two Xcode projects + a workspace**, not a Swift package, and its modules are coarser than the split below: one core `EssentialFeed.framework` holds domain, API, cache, presentation **and** infrastructure (it imports CoreData), separated only by folders (`Feed API/`, `Feed Cache/`, `Shared API Infra/`…); `EssentialFeediOS` holds the UIKit layer and `EssentialApp` the Composition Root. That core is UI-free, not Foundation-only. The finer SPM split in this file is **the skill's recommendation** — it turns the case study's folder boundaries into compiler-enforced ones. Use it when you want package-based modularity; the dependency rules are identical either way.

---

## Onion architecture

```
             ┌──────────────── frameworks ────────────────┐
             │  ┌──────── infrastructure adapters ──────┐ │
             │  │  ┌──── use cases + presentation ────┐ │ │
             │  │  │   domain models + boundaries     │ │ │   ← no outgoing arrows
             │  │  └──────────────────────────────────┘ │ │
             │  └───────────────────────────────────────┘ │
             └───────────────────────────────────────────┘
```

Two rules, and only two:

1. **Inner layers know nothing about outer layers.** The centre — domain models and service
   interfaces — has **no outgoing arrows at all**.
2. **An outer layer knows only its *direct* inner layer.** And sibling slices (API, Cache,
   Presentation) never know about each other.

The dividing line that matters in practice is **platform-agnostic vs platform-specific**:
everything above infrastructure is reusable on macOS, watchOS, tvOS and iPadOS. `URLSession` and
CoreData are Apple-only, so pushing them into an infrastructure ring is what would let the core
run somewhere else entirely.

**The test is "peeling the onion":** you should be able to take a whole vertical slice — the cache
layer plus its CoreData infrastructure — off the diagram, and replace or delete it without
touching anything else. If no arrow snaps, the boundaries are real. If one does, they were
decoration.

---

## Horizontal vs vertical slicing

| | Slices by | Produces |
|---|---|---|
| **Horizontal** | Layer | `Feed`, `FeedAPI`, `FeedCache`, `FeedPresentation`, `FeediOS` |
| **Vertical** | Feature | `Feed` framework, `Login` framework, `Comments` framework |

They compose: slice **vertically by feature**, and inside each feature slice **horizontally by
layer**. The app target composes the vertical slices; the Composition Root is the only place that
imports everything, which is exactly how the modules stay agnostic of each other and still combine.

The layout in this file is one feature sliced horizontally. Add a second feature and it becomes a
vertical slice beside it — with its own API, cache and UI targets, and no arrow between the two.

> Separate modules are what let separate teams work in parallel: a feed team and a comments team
> never touch the same files, and the root composes both.

---

## How physically separated should modules be?

Folders give you **virtual** modularity — the compiler will not stop you crossing a boundary, so
it takes discipline. Physical separation forces `import` statements, and the compiler enforces the
boundary for you. That enforcement has a price, so escalate only when you need it:

| # | Setup | Use when | Cost |
|---|---|---|---|
| **1** | **Monolith** — one project, one app target, folders only | Small apps, prototypes, code you'll discard | No compiler enforcement; tests run through the simulator, so slower |
| **2** | **Feature vertical slicing** — one framework target per feature, one project | You want features physically separated, layers virtually | Target maintenance |
| **3** | **Horizontal slicing** — a framework target per layer (`FeedAPI`, `FeedCache`…) | You want compiler-enforced layer boundaries | Every extraction forces `import`s and linking |
| **4** | **Horizontal within vertical** — each feature has its own layered stack | Several features, each non-trivial, features don't know each other | The most targets to maintain |
| **5** | **One project per feature**, combined in an app workspace | Larger apps, multiple teams, still one repo | Workspace + scheme management |
| **6** | **Packages / separate repositories** | You genuinely share modules **across repos** | Highest — SwiftPM/CocoaPods/Carthage/submodules, version pinning, cross-repo releases |

Guidance:

- **Start simple; split when needed.** Breaking into physically separated modules increases the
  maintenance you and your team carry. Do it when you have the problem it solves.
- **Keep everything in one repository when you can.** A monorepo is by far the easiest way to
  maintain a modular app under source control — and it is what makes cross-module refactors cheap.
- **The decision is driven by team structure**, not by taste: feature teams, remote teams and
  release cadence all influence it.
- Before splitting a target to speed up the test loop, try **controlling which tests run per
  scheme** — that is often enough.

An interfaces-only shared module is legitimate, but note its cost: every change to it forces a
recompile and redeploy of every module that depends on it. **A shared module with many incoming
arrows is a bottleneck**; prefer letting the Composition Root do the sharing, since nothing
depends on it.

> Duplicating a small *interface* across two modules is often cheaper than coupling them. Two
> modules each declaring the one-method `HTTPClient` they need is Interface Segregation, not
> duplication — and it lets one of them grow a `POST` without affecting the other. Share the
> *implementation* instead: put the concrete client in an infrastructure module that depends on
> both protocols. Look at the arrow direction — infrastructure depends on the API modules, never
> the reverse.

---

## Module Layout

The skill's recommended layout for one feature (`Feed`), sliced horizontally. Target names, folder names and product names match
one-to-one — no `path:` overrides to remember:

```
MyApp/
├── MyApp.xcodeproj           # the app — consumes the local package (see below)
├── App/                      # app target sources — Composition Root
│   ├── MyAppApp.swift        #   @main App (SwiftUI) — or AppDelegate/SceneDelegate (UIKit)
│   ├── CompositionRoot/
│   │   ├── FeedService.swift
│   │   ├── FeedUIComposer.swift
│   │   ├── Adapters/         #   presentation adapters (LoadResourcePresentationAdapter…)
│   │   └── Proxies/          #   WeakRefVirtualProxy and friends
│   └── Resources/            #   app icon, launch screen, Info.plist
└── Modules/                  # ONE local package, many targets
    ├── Package.swift
    ├── Sources/
    │   ├── Feed/                 # Domain models + boundary contracts (Foundation only)
    │   ├── FeedAPI/              # Endpoint enums, static mappers (Foundation only)
    │   ├── FeedCache/            # Store protocols, local models, loaders, cache policy (Foundation only)
    │   ├── FeedCacheInfra/       # CoreData/InMemory store implementations (imports CoreData)
    │   ├── FeedPresentation/     # Presenters, view models, localized strings (Foundation only)
    │   ├── FeediOS/              # UIKit views, cell controllers (imports UIKit)
    │   └── FeedSwiftUI/          # SwiftUI views (imports SwiftUI) — ship FeediOS, FeedSwiftUI, or both
    └── Tests/
        ├── FeedTests/            # Unit tests: domain, API, cache, presentation
        ├── FeedCacheInfraTests/  # Integration tests with real CoreData
        ├── FeediOSTests/         # Snapshot tests
        └── FeedEndToEndTests/    # API end-to-end tests (CI-only scheme)
```

**Key rule**: `Feed`, `FeedAPI`, `FeedCache` and `FeedPresentation` import only Foundation.
`FeedCacheInfra` imports CoreData. `FeediOS` imports UIKit; `FeedSwiftUI` imports SwiftUI. The app
target imports everything — it is the only place where all modules converge.

**Why `FeedCacheInfra` is separate**: infrastructure implementations (`CoreDataFeedStore`,
`InMemoryFeedStore`) depend on specific frameworks (CoreData). Separating them from `FeedCache`
(which defines protocols and business logic) keeps the cache layer framework-free and testable
with in-memory doubles.

**Why `FeedPresentation` is its own target**: presenters are platform-agnostic by design — that is
the whole argument for MVP's cross-platform presentation layer. Folding them into `Feed` would
make the domain target carry localized string tables; folding them into `FeediOS` would tie them
to UIKit. A separate target keeps both boundaries compiler-enforced (and lets `FeedSwiftUI` and
`FeediOS` share the same presenters).

### How the app consumes the package

The app is an Xcode project; the modules are a **local package** inside the same repo:

1. In Xcode: **File → Add Package Dependencies… → Add Local…** and select `Modules/` (this stores
   a relative `XCLocalSwiftPackageReference` in the project — no versioning, always live), then add
   the library products to the app target's *Frameworks, Libraries, and Embedded Content*.
2. From another *package* (rare — e.g. a dev-app package), the same thing is
   `.package(path: "../Modules")`.
3. Test plans/schemes: the app's scheme runs the app + acceptance tests; add the package's test
   targets to a `CI` test plan so `xcodebuild test` covers both. Editing the package and the app
   in one workspace works out of the box — a local package needs no "Open in new window" dance.

Local-package modularity keeps level-3/4 boundaries (compiler-enforced imports) at monorepo cost —
no version pinning, no separate releases, cross-module refactors in one commit.

### Inside a module

Group by **role within the layer**, mirroring the case study's proven groups — not by type
(`Protocols/`, `Extensions/` folders are how related code gets scattered):

```
Sources/FeedCache/
├── FeedStore.swift               # the boundary protocol(s)
├── LocalFeedLoader.swift         # the use case (split per CQS when it grows)
├── LocalFeedImage.swift          # the layer's data-transfer representation
├── FeedCachePolicy.swift         # pure policy — deterministic core
└── Helpers/                      # layer-private helpers only

Tests/FeedTests/Feed Cache/
├── CacheFeedUseCaseTests.swift   # one suite per use case…
├── LoadFeedFromCacheUseCaseTests.swift
├── ValidateFeedCacheUseCaseTests.swift
└── Helpers/                      # FeedStoreSpy, shared factories
```

Test folders mirror production groups one-to-one, and suites are named for **use cases**, not for
classes — three suites over `LocalFeedLoader` is correct, one `LocalFeedLoaderTests` is not.

---

## Package.swift Example

```swift
// swift-tools-version: 6.2
// NOTE: `.v26` mirrors the case study. Use the app's real deployment target instead — never raise
// it to unlock an API. The `.v26` constants and `.defaultIsolation` require tools 6.2; on older
// targets use plain `Task` where this skill shows `Task.immediate` (concurrency-at-boundaries.md).
import PackageDescription

let package = Package(
    name: "Modules",
    platforms: [.iOS(.v26), .macOS(.v26)],
    products: [
        .library(name: "Feed", targets: ["Feed"]),
        .library(name: "FeedAPI", targets: ["FeedAPI"]),
        .library(name: "FeedCache", targets: ["FeedCache"]),
        .library(name: "FeedCacheInfra", targets: ["FeedCacheInfra"]),
        .library(name: "FeedPresentation", targets: ["FeedPresentation"]),
        .library(name: "FeediOS", targets: ["FeediOS"]),
        .library(name: "FeedSwiftUI", targets: ["FeedSwiftUI"]),
    ],
    targets: [
        // Feature layer — Foundation only
        .target(name: "Feed"),

        // API layer — depends on Feature for domain models
        .target(name: "FeedAPI", dependencies: ["Feed"]),

        // Cache layer — depends on Feature for domain models (Foundation only)
        .target(name: "FeedCache", dependencies: ["Feed"]),

        // Cache infrastructure — CoreData/InMemory implementations
        .target(
            name: "FeedCacheInfra",
            dependencies: ["FeedCache"],
            resources: [
                .process("CoreData/FeedStore.xcdatamodeld"),
            ]
        ),

        // Presentation — platform-agnostic presenters + localized tables
        .target(
            name: "FeedPresentation",
            dependencies: ["Feed"],
            resources: [
                .process("Feed.strings"),
            ]
        ),

        // UI layers — pick one, or ship both against the same presenters
        .target(
            name: "FeediOS",
            dependencies: ["Feed", "FeedPresentation"],
            resources: [
                .process("Feed UI/Feed.storyboard"),
            ]
        ),
        .target(name: "FeedSwiftUI", dependencies: ["Feed", "FeedPresentation"]),

        // Unit tests — domain, API, cache, presentation
        .testTarget(
            name: "FeedTests",
            dependencies: ["Feed", "FeedAPI", "FeedCache", "FeedPresentation"]
        ),

        // Cache infrastructure integration tests (real CoreData)
        .testTarget(name: "FeedCacheInfraTests", dependencies: ["FeedCacheInfra", "FeedCache"]),

        // Snapshot tests
        .testTarget(name: "FeediOSTests", dependencies: ["FeediOS", "FeedPresentation"]),

        // API end-to-end tests — CI-only scheme
        .testTarget(name: "FeedEndToEndTests", dependencies: ["Feed", "FeedAPI"]),
    ]
)
```

With names, folders and products aligned, no target needs a `path:` override — `Sources/<name>`
and `Tests/<name>` are the defaults.

> Test targets depend on the modules under test and `import` them plainly — **never** `@testable import`. Tests exercise each module through its public API only; if a test can't reach what it needs, expose a public seam rather than widening access. The App target is the exception: nothing depends on the Composition Root, so its types need not be `public` and its own tests may use `@testable`. See [testability-and-seams.md](testability-and-seams.md#0-test-through-the-public-api-no-testable-import).

---

## Adding a second feature — what slicing looks like on disk

The layout above is one vertical slice. Here is the same package with a second feature
(`ImageComments`) and the cross-cutting modules that emerge — mirroring the case study's proven
groups (`Shared API`, `Shared Presentation`):

```
Modules/Sources/
├── Feed/                     ├── ImageComments/            # each feature: its own vertical slice,
├── FeedAPI/                  ├── ImageCommentsAPI/         # horizontally layered inside
├── FeedCache/                ├── ImageCommentsPresentation/
├── FeedCacheInfra/           ├── ImageCommentsiOS/
├── FeedPresentation/         │
├── FeediOS/                  │   # note: no ImageCommentsCache — a feature only
│                             │   # grows the layers it needs
├── SharedAPI/                # HTTPClient protocol, Paginated<Item> (Foundation only)
├── SharedAPIInfra/           # URLSessionHTTPClient (imports Foundation networking)
├── SharedPresentation/       # LoadResourcePresenter<Resource, View>, Resource*View models
└── SharediOS/                # ListViewController, CellController, shared UIKit helpers
```

The rules that keep this honest:

- **By default, feature slices don't import each other.** `Feed` and `ImageComments` are connected
  only in the Composition Root (navigation from a feed cell to its comments is composed there —
  neither module knows the other exists). This is a judgment call, not a law: comments are an
  extension of the feed, so an arrow like `ImageCommentsEndpoint.get(FeedImage)` would be
  acceptable. Full decoupling pays off mainly when separate teams own the features; until then,
  folder-level separation of decoupled types is enough, and extracting them later is cheap (007/004).
- **`Shared*` modules earn their existence by generalization, not by anticipation.** They hold
  code that *two* features already demanded (the generalization arc in
  `presentation-patterns.md`) — never code that "might be shared someday".
- **Watch the incoming arrows.** In the target design, feature modules (`FeedAPI`,
  `ImageCommentsPresentation`, `ImageCommentsiOS`) do **not** import `Shared*`. They expose pure
  mappers, `title` + `map` presentation functions and platform-protocol conformances
  (`UITableViewDataSource`/`Delegate`/`Prefetching`), and only the Composition Root imports both the
  features and the shared generics (`LoadResourcePresenter`, `ListViewController`, `HTTPClient`).
  A feature→shared arrow is an acceptable intermediate step, not the goal — "every time we change
  the shared module, you may break every other module that depends on it" (007/002). If a shared
  module starts accumulating feature-specific knowledge, that code was never shared — move it back.
- A feature adds only the layers it needs — `ImageComments` has no cache, so there is no empty
  `ImageCommentsCache` target "for symmetry".

The resulting arrows — features point only at their own domain; the shared modules are pulled in
by the App target:

```
ImageCommentsAPI ──────────► ImageComments          SharedAPIInfra ──► SharedAPI
ImageCommentsPresentation ─► ImageComments          SharedPresentation
ImageCommentsiOS ──────────► ImageComments,         SharediOS
                             ImageCommentsPresentation

App Target (Composition Root) ──► every feature module + every Shared* module
```

## Where the Composition Root lives

Default: **in the app target**, as plain folders (`App/CompositionRoot/`). Nothing may depend on
the Composition Root, so it gains nothing from being a package target — and leaving it in the app
target lets its types stay `internal` and lets acceptance tests use `@testable import` (the one
sanctioned place — see `testability-and-seams.md`).

Promote it to a dedicated `CompositionRoot` package target only when something real forces it —
two app targets (App Clip, widget, watch app) that must share wiring, or a dev-harness app that
boots slices of the graph. Then it is the **one** target that imports every module, the app
target shrinks to `@main` plus lifecycle, and its composers must become `public`. The dependency
rule is unchanged: everything points inward, nothing points at the root.

## Static vs dynamic linking

SwiftPM builds library targets as **static** by default, and static is what you want here: with
seven-plus targets per feature, dynamic frameworks would each cost dyld work at launch. Reach for
`.library(type: .dynamic)` only when one binary must be shared across independently-loaded
products (app + extensions embedding the same large module) — and consider **mergeable
libraries** (Xcode 15+) when you want dynamic-in-Debug, static-in-Release behaviour. The module
*boundaries* are identical either way; linking is a deployment detail, so decide it by
launch-time and app-size measurements, not by architecture.

---

## Build Settings — Swift 6 Strict Concurrency

| Setting | SwiftPM (`Package.swift`) | Xcode (`.pbxproj`) | Why it matters |
|---|---|---|---|
| Language mode | `swift-tools-version: 6.0`+ (6.2 for `.v26`/`.defaultIsolation`) | Swift Language Version = 6 | Enables strict concurrency by default |
| Strict concurrency | Enabled by default in Swift 6 | `SWIFT_STRICT_CONCURRENCY = complete` | Compile-time data-race safety |
| Upcoming features | `.enableUpcomingFeature("...")` in `swiftSettings` | `SWIFT_UPCOMING_FEATURE_*` | Opt-in to future language changes |

For targets not yet on the Swift 6 language mode, enable strict concurrency *warnings* first:

```swift
// In Package.swift targets
.target(
    name: "Feed",
    swiftSettings: [
        .swiftLanguageMode(.v5),
        .enableUpcomingFeature("StrictConcurrency")
    ]
)
```

(The older `.enableExperimentalFeature("StrictConcurrency=complete")` spelling still works on
current toolchains, but unknown *experimental* feature names are silently ignored — if a future
compiler drops the name, you lose the checking without a single diagnostic. The upcoming-feature
spelling fails loudly instead; verified on Swift 6.3.3.)

Enable on **all targets** — feature, API, cache, infra, presentation, UI, and App. Inconsistent
settings across modules hide data-race issues at module boundaries.

### Multi-platform support as boundary enforcement

Compiling the core (Feature/API/Cache/Presentation) for **both macOS and iOS** is the cheapest
enforcement of zero UIKit imports there is: the second platform fails to build the moment a UI
framework leaks into a framework-free layer. It is a compiler-enforced version of the "delete the
import and see if it still compiles" check.

Making it work:

- If a target's code has no platform dependency, list **all** the platforms it supports — and do
  the same for its **test targets**, not just the production ones.
- **Run the tests on every platform you support.** Behaviour that is correct on one platform is
  occasionally wrong on another, and you only find that by running there.
- The moment a platform-specific component (a `UIViewController` subclass) lands in a shared
  target, that target loses macOS support instantly. That component belongs in the platform-specific
  module.
- **`platforms:` sets minimum versions, it does not filter targets.** SwiftPM has no per-target
  platform list: building the whole package for macOS fails on every UIKit target. A macOS build
  works only through a scheme (or `swift build --target`) that selects the platform-independent
  targets and their tests. Never claim a manifest restricts a target to iOS.
- **CI needs one scheme per platform** — you cannot put an iOS-only test target in a scheme that
  also builds for macOS. Share `CI_macOS` and `CI_iOS` schemes and run both, so
  platform-independent tests run everywhere and platform-specific tests run only where they apply.

The cost is honest: more platforms means slower CI and higher build-server bills. Take it when the
boundary enforcement (or genuine multi-platform reach) is worth it.

---

## Resource Declarations

CoreData models and localization strings must be declared as SPM resources:

```swift
// CoreData model
resources: [
    .process("CoreData/FeedStore.xcdatamodeld"),
]

// Localization strings
resources: [
    .process("Shared Presentation/Shared.strings"),
]

// Storyboards (UIKit)
resources: [
    .process("Feed UI/Feed.storyboard"),
]
```

Access with `Bundle.module` in SPM:

```swift
// Instead of Bundle(for: Self.self)
NSLocalizedString("GENERIC_CONNECTION_ERROR",
                  tableName: "Shared",
                  bundle: .module,
                  comment: "")
```

---

## Dependency Graph

```
Feed (Foundation)
    |
    +-- FeedAPI (Foundation)
    +-- FeedCache (Foundation)
    |       |
    |       +-- FeedCacheInfra (CoreData)
    +-- FeedPresentation (Foundation)
            |
            +-- FeediOS (UIKit)
            +-- FeedSwiftUI (SwiftUI)
                    |
                    App Target (imports all — Composition Root)
```

Dependencies flow **inward**: UI -> Presentation -> Feature, Cache -> Feature, API -> Feature.
The `Feed` module depends on nothing. The App target depends on everything but is the thinnest
layer — just wiring.

---

## CI/CD GitHub Actions Example

```yaml
jobs:
  test:
    runs-on: macos-15
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: |
          xcodebuild test -scheme Feed \
            -destination 'platform=iOS Simulator,name=iPhone 16,OS=latest' \
            -enableThreadSanitizer YES \
            -resultBundlePath TestResults.xcresult
```

**Key flags / settings**:

| Flag / setting | Where | Why |
|---|---|---|
| `-enableThreadSanitizer YES` | CI `xcodebuild` invocation, **not** the shared scheme | Catches data races at `@Sendable`/`@MainActor` boundaries. Keep it out of the shared scheme — duplicating it there triggers a duplicate-path warning. |
| `-com.apple.CoreData.ConcurrencyDebug 1` | test scheme arguments | Hard-fails on any CoreData context-queue violation. |
| Randomized test order | test scheme | Surfaces hidden inter-test ordering/shared-state dependencies. |
| Code coverage enabled | test scheme | Required for coverage artifacts. |

End-to-end API tests get their own CI-only scheme so they never run in the fast dev loop.

---

## Guardrails

- Do not add UIKit/SwiftUI imports to feature, API, cache, or presentation targets
- Do not skip `FeedCacheInfra` — CoreData implementations belong in infrastructure, not in the cache layer
- Do not fold presenters into the domain or UI target — `FeedPresentation` is its own platform-agnostic target
- Do not mix test types in the same target — keep unit, integration, snapshot, and acceptance tests separate
- Do not use `Bundle(for:)` in SPM targets — use `Bundle.module`
- Do not split into physically separated modules before you have the problem it solves — start with folders and escalate
- Do not create a shared module that every other module depends on — let the Composition Root do the sharing; a feature→`Shared*` arrow is a stepping stone, not the target design
- Do not let feature modules import each other by default — cross-feature flows are composed in the root; a deliberate arrow between closely related features is a judgment call, worth avoiding mainly when separate teams own them
- Do not let a feature module depend on the Composition Root — that is a circular dependency waiting to happen
- Do not make library products `.dynamic` by default — static is the right default; measure first

## Verification

- [ ] `swift-tools-version` matches the APIs used (6.2 for the `.v26` platform constants)
- [ ] Platform pairs are coherent (`.iOS(.v26)` goes with `.macOS(.v26)` — SwiftPM won't check this for you)
- [ ] Every target in the layout tree exists in `Package.swift`, and vice versa — including each
      resource path a target `.process`es (show it in the tree; a missing path fails resolution)
- [ ] `Feed` target has zero dependencies
- [ ] `FeedAPI`, `FeedCache` and `FeedPresentation` depend only on `Feed`
- [ ] `FeedCacheInfra` depends on `FeedCache` and declares CoreData resources
- [ ] UI targets depend on `FeedPresentation`, never the reverse
- [ ] No feature module imports another feature module (or the arrow is a deliberate, documented judgment call)
- [ ] Feature modules don't import `Shared*` in the target design — only the App target does
- [ ] All targets build with `SWIFT_STRICT_CONCURRENCY = complete` — zero warnings
- [ ] CI runs with `-enableThreadSanitizer YES`
