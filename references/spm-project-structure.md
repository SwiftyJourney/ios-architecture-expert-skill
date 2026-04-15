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
    name: "EssentialFeed",
    platforms: [.iOS(.v17), .macOS(.v14)],
    products: [
        .library(name: "EssentialFeed", targets: ["EssentialFeed"]),
        .library(name: "EssentialFeedAPI", targets: ["EssentialFeedAPI"]),
        .library(name: "EssentialFeedCache", targets: ["EssentialFeedCache"]),
        .library(name: "EssentialFeedCacheInfra", targets: ["EssentialFeedCacheInfra"]),
        .library(name: "EssentialFeediOS", targets: ["EssentialFeediOS"]),
    ],
    targets: [
        // Feature layer — Foundation only
        .target(
            name: "EssentialFeed",
            dependencies: [],
            path: "Sources/Feature"
        ),

        // API layer — depends on Feature for domain models
        .target(
            name: "EssentialFeedAPI",
            dependencies: ["EssentialFeed"],
            path: "Sources/FeatureAPI"
        ),

        // Cache layer — depends on Feature for domain models (Foundation only)
        .target(
            name: "EssentialFeedCache",
            dependencies: ["EssentialFeed"],
            path: "Sources/FeatureCache"
        ),

        // Cache infrastructure — CoreData/InMemory implementations
        .target(
            name: "EssentialFeedCacheInfra",
            dependencies: ["EssentialFeedCache"],
            path: "Sources/FeatureCacheInfra",
            resources: [
                .process("CoreData/FeedStore.xcdatamodeld"),
            ]
        ),

        // iOS UI layer — depends on Feature for view models
        .target(
            name: "EssentialFeediOS",
            dependencies: ["EssentialFeed"],
            path: "Sources/FeatureiOS",
            resources: [
                .process("Feed UI/Feed.storyboard"),
                .process("Shared Presentation/Shared.strings"),
            ]
        ),

        // Unit tests
        .testTarget(
            name: "EssentialFeedTests",
            dependencies: ["EssentialFeed", "EssentialFeedAPI", "EssentialFeedCache"],
            path: "Tests/FeatureTests"
        ),

        // Cache infrastructure integration tests
        .testTarget(
            name: "EssentialFeedCacheInfraTests",
            dependencies: ["EssentialFeedCacheInfra", "EssentialFeedCache"],
            path: "Tests/FeatureCacheInfraTests"
        ),

        // Snapshot tests
        .testTarget(
            name: "EssentialFeediOSTests",
            dependencies: ["EssentialFeediOS", "EssentialFeed"],
            path: "Tests/FeatureiOSTests"
        ),
    ]
)
```

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
    name: "EssentialFeed",
    dependencies: [],
    swiftSettings: [
        .enableExperimentalFeature("StrictConcurrency=complete")
    ]
)
```

Enable on **all targets** — Feature, API, Cache, CacheInfra, iOS, and App. Inconsistent settings across modules hide data-race issues at module boundaries.

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
EssentialFeed (Foundation)
    |
    +-- EssentialFeedAPI (Foundation)
    +-- EssentialFeedCache (Foundation)
    |       |
    |       +-- EssentialFeedCacheInfra (CoreData)
    +-- EssentialFeediOS (UIKit)
            |
            App Target (imports all — Composition Root)
```

Dependencies flow **inward**: UI -> Feature, Cache -> Feature, API -> Feature. The Feature module depends on nothing. The App target depends on everything but is the thinnest layer — just wiring.

---

## CI/CD GitHub Actions Example

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: macos-15
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_16.app

      - name: Run tests
        run: |
          xcodebuild test \
            -scheme EssentialFeed \
            -destination 'platform=iOS Simulator,name=iPhone 16,OS=latest' \
            -enableThreadSanitizer YES \
            -resultBundlePath TestResults.xcresult

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: TestResults
          path: TestResults.xcresult
```

**Key flags**:

- `-enableThreadSanitizer YES`: Catches data races, especially important with `@Sendable` and `@MainActor` boundaries
- `-resultBundlePath`: Captures detailed test results for CI artifacts
- `macos-15` + Xcode 16: Ensures Swift 6 / iOS 18 compatibility

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
