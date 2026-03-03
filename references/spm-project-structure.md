# SPM Project Structure

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

**Key rule**: `Feature` and `FeatureAPI`/`FeatureCache` import only Foundation. `FeatureiOS` imports UIKit. `FeatureSwiftUI` imports SwiftUI. The app target imports everything — it is the only place where all modules converge.

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

        // Cache layer — depends on Feature for domain models
        .target(
            name: "EssentialFeedCache",
            dependencies: ["EssentialFeed"],
            path: "Sources/FeatureCache",
            resources: [
                .process("Infrastructure/CoreData/FeedStore.xcdatamodeld"),
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

## Resource Declarations

CoreData models and localization strings must be declared as SPM resources:

```swift
// CoreData model
resources: [
    .process("Infrastructure/CoreData/FeedStore.xcdatamodeld"),
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
    ↑
    ├── EssentialFeedAPI (Foundation)
    ├── EssentialFeedCache (Foundation + CoreData)
    └── EssentialFeediOS (UIKit)
            ↑
            App Target (imports all — Composition Root)
```

Dependencies flow **inward**: UI → Feature, Cache → Feature, API → Feature. The Feature module depends on nothing. The App target depends on everything but is the thinnest layer — just wiring.

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
