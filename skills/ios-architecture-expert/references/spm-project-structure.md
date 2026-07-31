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
- [Module Layout](#module-layout)
- [Package.swift Example](#packageswift-example)
- [Build Settings](#build-settings--swift-6-strict-concurrency)
- [Resource Declarations](#resource-declarations)
- [Dependency Graph](#dependency-graph)
- [CI/CD](#cicd-github-actions-example)

> The case study itself ships as **two Xcode projects + a workspace**, not a Swift package. This file is the recommended SPM-equivalent of that layout — use it when you want package-based modularity; the boundary rules are identical either way.

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

| Level | Setup | Use when | Cost |
|---|---|---|---|
| **A** | One project, one app target, folders only | Small apps, prototypes, throwaway projects | No enforcement; tests must run through the simulator; slower builds |
| **B** | One project, one framework target per feature | You want features physically separated, layers virtually | Target maintenance |
| **C** | One project per feature, composed by the app target | Larger apps, multiple teams | Workspace + scheme management |
| **D** | Separate repositories | You genuinely share code **across** repos | Highest — needs SwiftPM/Carthage/submodules, version pinning, cross-repo releases |

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

```
MyApp/
├── Package.swift
├── Sources/
│   ├── Feature/              # Domain models + use-case protocols (Foundation only)
│   ├── FeatureAPI/           # HTTPClient protocol, endpoint enums, static mappers
│   ├── FeatureCache/         # Store protocols, local models, loaders, cache policy
│   ├── FeatureCacheInfra/    # CoreData/InMemory store implementations
│   ├── FeatureiOS/           # UIKit views, cell controllers (imports UIKit)
│   └── FeatureSwiftUI/       # SwiftUI views (imports SwiftUI)
├── Tests/
│   ├── FeatureTests/         # Unit tests for domain, API, cache, presentation
│   ├── FeatureCacheInfraTests/  # Integration tests with real CoreData
│   ├── FeatureiOSTests/      # Snapshot tests
│   └── FeatureEndToEndTests/ # API end-to-end tests
└── App/                      # Xcode app target — Composition Root
    ├── CompositionRoot/
    │   ├── FeedService.swift
    │   ├── FeedUIComposer.swift
    │   ├── Adapters/
    │   └── Proxies/
    └── SceneDelegate.swift
```

**Key rule**: `Feature` and `FeatureAPI`/`FeatureCache` import only Foundation. `FeatureCacheInfra` imports CoreData. `FeatureiOS` imports UIKit. `FeatureSwiftUI` imports SwiftUI. The app target imports everything — it is the only place where all modules converge.

**Why `FeatureCacheInfra` is separate**: Infrastructure implementations (`CoreDataFeedStore`, `InMemoryFeedStore`) depend on specific frameworks (CoreData). Separating them from `FeatureCache` (which defines protocols and business logic) keeps the cache layer framework-free and testable with in-memory doubles.

---

## Package.swift Example

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "Feed",
    // NOTE: the concurrency patterns in this skill (e.g. `Task.immediate`) need a Swift 6.2-era
    // toolchain and a recent deployment target. Do not pair them with .iOS(.v17) — bump the
    // platforms to match, or fall back to plain `Task` (see concurrency-at-boundaries.md).
    platforms: [.iOS(.v26), .macOS(.v15)],
    products: [
        .library(name: "Feed", targets: ["Feed"]),
        .library(name: "FeedAPI", targets: ["FeedAPI"]),
        .library(name: "FeedCache", targets: ["FeedCache"]),
        .library(name: "FeedCacheInfra", targets: ["FeedCacheInfra"]),
        .library(name: "FeediOS", targets: ["FeediOS"]),
    ],
    targets: [
        // Feature layer — Foundation only
        .target(
            name: "Feed",
            dependencies: [],
            path: "Sources/Feature"
        ),

        // API layer — depends on Feature for domain models
        .target(
            name: "FeedAPI",
            dependencies: ["Feed"],
            path: "Sources/FeatureAPI"
        ),

        // Cache layer — depends on Feature for domain models (Foundation only)
        .target(
            name: "FeedCache",
            dependencies: ["Feed"],
            path: "Sources/FeatureCache"
        ),

        // Cache infrastructure — CoreData/InMemory implementations
        .target(
            name: "FeedCacheInfra",
            dependencies: ["FeedCache"],
            path: "Sources/FeatureCacheInfra",
            resources: [
                .process("CoreData/FeedStore.xcdatamodeld"),
            ]
        ),

        // iOS UI layer — depends on Feature for view models
        .target(
            name: "FeediOS",
            dependencies: ["Feed"],
            path: "Sources/FeatureiOS",
            resources: [
                .process("Feed UI/Feed.storyboard"),
                .process("Shared Presentation/Shared.strings"),
            ]
        ),

        // Unit tests
        .testTarget(
            name: "FeedTests",
            dependencies: ["Feed", "FeedAPI", "FeedCache"],
            path: "Tests/FeatureTests"
        ),

        // Cache infrastructure integration tests
        .testTarget(
            name: "FeedCacheInfraTests",
            dependencies: ["FeedCacheInfra", "FeedCache"],
            path: "Tests/FeatureCacheInfraTests"
        ),

        // Snapshot tests
        .testTarget(
            name: "FeediOSTests",
            dependencies: ["FeediOS", "Feed"],
            path: "Tests/FeatureiOSTests"
        ),
    ]
)
```

> Test targets depend on the modules under test and `import` them plainly — **never** `@testable import`. Tests exercise each module through its public API only; if a test can't reach what it needs, expose a public seam rather than widening access. The App target is the exception: nothing depends on the Composition Root, so its types need not be `public` and its own tests may use `@testable`. See [testing-strategy.md](testing-strategy.md#0-test-through-the-public-api-no-testable-import).

---

## Build Settings — Swift 6 Strict Concurrency

| Setting | SwiftPM (`Package.swift`) | Xcode (`.pbxproj`) | Why it matters |
|---|---|---|---|
| Language mode | `swift-tools-version: 6.0` | Swift Language Version = 6 | Enables strict concurrency by default |
| Strict concurrency | Enabled by default in Swift 6 | `SWIFT_STRICT_CONCURRENCY = complete` | Compile-time data-race safety |
| Upcoming features | `.enableUpcomingFeature("...")` in `swiftSettings` | `SWIFT_UPCOMING_FEATURE_*` | Opt-in to future language changes |

For projects not yet on Swift 6, enable strict concurrency incrementally:

```swift
// In Package.swift targets
.target(
    name: "Feed",
    dependencies: [],
    swiftSettings: [
        .enableExperimentalFeature("StrictConcurrency=complete")
    ]
)
```

Enable on **all targets** — Feature, API, Cache, CacheInfra, iOS, and App. Inconsistent settings across modules hide data-race issues at module boundaries.

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
    +-- FeediOS (UIKit)
            |
            App Target (imports all — Composition Root)
```

Dependencies flow **inward**: UI -> Feature, Cache -> Feature, API -> Feature. The Feature module depends on nothing. The App target depends on everything but is the thinnest layer — just wiring.

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

- Do not add UIKit/SwiftUI imports to Feature, API, or Cache targets
- Do not skip `FeatureCacheInfra` — CoreData implementations belong in infrastructure, not in the cache layer
- Do not mix test types in the same target — keep unit, integration, snapshot, and acceptance tests separate
- Do not use `Bundle(for:)` in SPM targets — use `Bundle.module`
- Do not split into physically separated modules before you have the problem it solves — start with folders and escalate
- Do not create a shared module that every other module depends on — let the Composition Root do the sharing
- Do not let a feature module depend on the Composition Root — that is a circular dependency waiting to happen

## Verification

- [ ] `swift-tools-version: 6.0` (or strict concurrency enabled via `swiftSettings`)
- [ ] `Feature` target has zero dependencies
- [ ] `FeatureAPI` and `FeatureCache` depend only on `Feature`
- [ ] `FeatureCacheInfra` depends on `FeatureCache` and declares CoreData resources
- [ ] All targets build with `SWIFT_STRICT_CONCURRENCY = complete` — zero warnings
- [ ] CI runs with `-enableThreadSanitizer YES`
