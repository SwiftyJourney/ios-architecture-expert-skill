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
- [Module Layout](#module-layout)
- [Package.swift Example](#packageswift-example)
- [Build Settings](#build-settings--swift-6-strict-concurrency)
- [Resource Declarations](#resource-declarations)
- [Dependency Graph](#dependency-graph)
- [CI/CD](#cicd-github-actions-example)

> The case study itself ships as **two Xcode projects + a workspace**, not a Swift package. This file is the recommended SPM-equivalent of that layout — use it when you want package-based modularity; the boundary rules are identical either way.

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

> Test targets depend on the modules under test and `import` them plainly — **never** `@testable import`. Tests exercise each module through its public API only; if a test can't reach what it needs, expose a public seam rather than widening access. See [testing-strategy.md](testing-strategy.md#0-test-through-the-public-api-no-testable-import).

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

> Compiling the core package (Feature/API/Cache/Presentation) for **both macOS and iOS** is the cheapest enforcement of zero UIKit imports — a second platform fails to build the moment a UI framework leaks into a framework-free layer.

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

## Verification

- [ ] `swift-tools-version: 6.0` (or strict concurrency enabled via `swiftSettings`)
- [ ] `Feature` target has zero dependencies
- [ ] `FeatureAPI` and `FeatureCache` depend only on `Feature`
- [ ] `FeatureCacheInfra` depends on `FeatureCache` and declares CoreData resources
- [ ] All targets build with `SWIFT_STRICT_CONCURRENCY = complete` — zero warnings
- [ ] CI runs with `-enableThreadSanitizer YES`
