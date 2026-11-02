# Architecture Layers — Code Reference

Use this when:
- You need to define a domain model, use-case protocol, or layer boundary
- You need to understand what imports are allowed in each layer
- You need code patterns for API mappers, cache layer, or presentation
- You need UIKit or SwiftUI UI layer patterns

Skip this file if:
- You need Composition Root wiring. Use `composition-root.md`.
- You need adapter/proxy patterns. Use `adapters-and-proxies.md`.
- You need concurrency annotations rationale. Use `concurrency-at-boundaries.md`.
- You need the *reasoning* for a boundary rather than the code. Use `design-principles.md`.

Jump to:
- [Three kinds of logic](#three-kinds-of-logic--the-framing)
- [Do you even need a persistence layer?](#do-you-even-need-a-persistence-layer)
- [Feature Layer](#feature-layer-foundation-only)
- [API Layer](#api-layer-foundation-only)
- [Cache Layer](#cache-layer-foundation-only)
- [Presentation Layer](#presentation-layer-foundation-only)
- [UI Layer — UIKit](#ui-layer--uikit)
- [UI Layer — SwiftUI](#ui-layer--swiftui-equivalent)

> **"Foundation only" is a per-layer import rule, not a description of the case study's modules.**
> In the case study, Feature, API, Cache and Presentation are *folders* inside one
> `EssentialFeed.framework` that also holds the CoreData and URLSession infrastructure — UI-free,
> not Foundation-only — next to `EssentialFeediOS` (UIKit) and `EssentialApp`. One SPM target per
> layer (`spm-project-structure.md`) is this skill's stricter recommendation. The code below is a
> worked example: map the names onto your own types.

---

## Three kinds of logic — the framing

The layer table in `SKILL.md` is a *file layout*. The rule that decides what goes where is
narrower: every line of code is one of three things.

| Kind | Example here | Deterministic? |
|---|---|---|
| **Application-specific** (use case / interactor / control boundary) | `LocalFeedLoader` — delete then insert with a timestamp | No — orchestrates side effects |
| **Application-agnostic** (business model / policy) | `FeedCachePolicy` — "expired" is a rule, not an app feature | Yes |
| **Framework** (infrastructure) | `CoreDataFeedStore`, `URLSessionHTTPClient` | No — *is* the side effect |

This is why the cache policy is a separate type from the loader, why the store contains no
conditional logic, and why models take injected data rather than injected functions. Full argument
and the diagnostics that go with it: [design-principles.md](design-principles.md#three-kinds-of-logic).

---

## Do you even need a persistence layer?

Before building a cache layer, check whether the platform already gives you one. The URL Loading
System caches HTTP/HTTPS responses in memory and on disk via `URLCache` — you can get that for
free by configuring a `URLSessionConfiguration`:

```swift
let cache = URLCache(memoryCapacity: 10_000_000, diskCapacity: 50_000_000)
let configuration = URLSessionConfiguration.default
configuration.urlCache = cache
configuration.requestCachePolicy = .returnCacheDataElseLoad
let session = URLSession(configuration: configuration)
```

If that satisfies the requirement, the store protocol, the local model, the cache policy, the
local loader and the fallback composition **all go away**.

| `URLCache` fits | `URLCache` is the wrong tool |
|---|---|
| Static-ish responses that change rarely | Offline *actions* (queue a like, upload later) |
| Small payloads | Anything needing your own expiry policy or eviction rules |
| Server sends correct `Cache-Control` | Data you must query, mutate or relate |

Caveats worth knowing: a response is cached only if it is an HTTP(S) request, the response
succeeded, it wasn't itself served from cache, both the session and request policies allow it, the
server's headers allow it, and the payload is small relative to the disk cache. Replacing
`URLCache.shared` mutates global state, so do it as early as possible in the app lifecycle — or,
better, inject your own session and never touch the shared one.

> The two are not exclusive: a custom persistence strategy can sit alongside `URLCache`. But when
> the app defines its own caching policy, use `.ephemeral` for the session so `URLCache` does not
> silently duplicate it — as this codebase does (`composition-root.md`).

---

## Feature Layer (Foundation only)

Domain models are value types. Boundaries are function types or, when earned, protocols. No framework imports.

```swift
// Domain model — struct, Sendable, and Equatable or Hashable
// (FeedImage is Hashable; ImageComment is only Equatable, Sendable — Hashable only when needed)
public struct FeedImage: Hashable, Sendable {
    public let id: UUID
    public let description: String?
    public let location: String?
    public let url: URL
}

// Store protocol — image data persistence (several cohesive methods, two real
// implementations: CoreData + InMemory)
public protocol FeedImageDataStore {
    func insert(_ data: Data, for url: URL) throws
    func retrieve(dataForURL url: URL) throws -> Data?
}
```

**Rules**: Data models (`FeedImage`) carry no behavior; business models and policies
(`FeedCachePolicy`) carry app-agnostic rules but no side effects and no state. A single-method
boundary is a function type unless it earns a protocol (below).

> **When does a boundary earn a protocol?** Not by counting strategies. A single-method boundary
> defaults to a function type (`() async throws -> R`) even with several implementations, because
> closures compose fallback, decoration and interception with plain code. Keep a protocol when the
> boundary has **several cohesive methods** (`FeedStore`, `FeedImageDataStore`), when a **named
> contract** matters to implementers (shared contract specs, external clients — `HTTPClient` is
> arguable but kept as the decorator/test-stub seam), or when **type-level conformance** carries
> meaning (`ResourceView` with its `associatedtype`). History: `FeedLoader` had remote, local and a
> fallback composite, and was still deleted (commit `fa5568b`) because Combine's universal
> abstractions made the named single-method protocol redundant. `FeedCache`,
> `FeedImageDataLoader` and `FeedImageDataCache` survive in the final code with one conformer each
> and no consumer — leftovers, not exemplars; `FeedService` calls the concrete `LocalFeedLoader`
> and `LocalFeedImageDataLoader` directly.

> **Infrastructure interfaces should be dumb.** `HTTPClient` and `FeedStore` exist only to fulfil
> commands sent by a controller type through the boundary — "fetch something from the cache",
> "download an image from a remote URL". No implementation of them ever contains a rule like "only
> insert if the timestamp is less than seven days old". Imagine how bad it would be if changing a
> database driver forced you to rewrite and re-migrate every business rule.

**Guardrails**:
- Do not make use-case protocols generic over their error type (`associatedtype Error`) — the constraint infects every consumer. The case study tried it and reverted within minutes; use a plain `Error` (or a locally-scoped typed `throws`).
- Do not add a production conformance (e.g. `Equatable` on a result/boundary enum) solely to satisfy a test assertion — pattern-match in an `expect(_:toCompleteWith:)` helper instead.

---

## API Layer (Foundation only)

Endpoint enums build URLs. Static mappers transform `Data + HTTPURLResponse` into domain models. Private `Decodable` types never leak.

```swift
// Endpoint — enum with associated values for pagination
public enum FeedEndpoint {
    case get(after: FeedImage? = nil)

    public func url(baseURL: URL) -> URL {
        switch self {
        case let .get(image):
            var components = URLComponents()
            components.scheme = baseURL.scheme
            components.host = baseURL.host
            components.path = baseURL.path + "/v1/feed"
            components.queryItems = [
                URLQueryItem(name: "limit", value: "10"),
                image.map { URLQueryItem(name: "after_id", value: $0.id.uuidString) },
            ].compactMap { $0 }
            return components.url!
        }
    }
}
```

- **Endpoint shape is a trade-off.** Adding a case or an associated value to an endpoint `enum`
  breaks every switch and call site — fine while only the Composition Root uses it. For a public or
  frequently changing endpoint API, prefer a struct with static factory methods. Model a cursor as
  an optional associated value (`get(after:)`), not a new case.
- **`components.url!` is deliberate here.** Building from a static, in-app base URL can only fail
  through a programmer error, so crash early and let tests catch it. In a public library, or from
  external input, return an optional and make callers handle it.

```swift
// Static mapper — the case study declares it `public final class FeedItemsMapper` with only
// static members. This skill recommends a caseless enum (an uninstantiable namespace) instead.
public enum FeedItemsMapper {
    private struct Root: Decodable {
        private let items: [RemoteFeedItem]

        private struct RemoteFeedItem: Decodable {
            let id: UUID
            let description: String?
            let location: String?
            let image: URL
        }

        var images: [FeedImage] {
            items.map { FeedImage(id: $0.id, description: $0.description,
                                  location: $0.location, url: $0.image) }
        }
    }

    public enum Error: Swift.Error {
        case invalidData
    }

    public static func map(_ data: Data, from response: HTTPURLResponse) throws -> [FeedImage] {
        guard response.isOK,
              let root = try? JSONDecoder().decode(Root.self, from: data) else {
            throw Error.invalidData
        }
        return root.images
    }
}
```

```swift
// HTTPClient protocol — async/await, returns tuple
public protocol HTTPClient {
    func get(from url: URL) async throws -> (Data, HTTPURLResponse)
}
```

**Rules**: Mappers expose only `static` methods. The skill's recommendation is a caseless `enum` — it cannot be instantiated, so it needs no private `init` (see `design-principles.md` on stateless namespaces); the case study's `public final class` mappers work the same but can be instantiated. Decodable types are `private`. Errors are typed enums. `HTTPClient` is a protocol, not `URLSession` directly.

- **Status-code contracts are per endpoint.** The feed mapper accepts exactly `200`; the image-comments mapper accepts any `2xx` (`(200...299).contains(...)`). Assert the exact accepted range per mapper — do not share one `isOK`.
- **Pagination is keyset-based** (a.k.a. the seek method), not limit/offset: `after_id=<last item's id>`.

| | Limit / offset | Keyset |
|---|---|---|
| Backend cost | Trivial (`LIMIT`/`OFFSET`) | Harder to implement |
| Random access to page N | Yes | No — next/previous only |
| Large offsets | Slow | Fast |
| Consistency while the data mutates | **Broken** — an insert shifts everything, producing duplicates and omissions | Stable — the key does not move |
| Variable page size mid-traversal | Not safe | Safe |
| Cacheable | Not really | **Yes** |

  Use limit/offset for small, short-lived, rarely-changing collections you will not cache (a
  currency list). Use keyset for infinite scroll and anything you cache — which is most feeds, and
  why consistent caching is the reason this codebase uses it, not the latency.
- **Do not define single-method loader protocols** (`FeedLoader`). The Composition Root composes `HTTPClient` + a static mapper into a plain `() async throws -> Resource` closure — the closure *is* the abstraction; the protocol just adds a name.

---

## Cache Layer (Foundation only)

Store protocols abstract persistence. Local models decouple cache representation from domain. Cache policy encapsulates business rules.

```swift
// Store protocol — synchronous throws API
public typealias CachedFeed = (feed: [LocalFeedImage], timestamp: Date)

public protocol FeedStore {
    func deleteCachedFeed() throws
    func insert(_ feed: [LocalFeedImage], timestamp: Date) throws
    func retrieve() throws -> CachedFeed?
}

// Local model — cache-layer representation
public struct LocalFeedImage: Equatable {
    public let id: UUID
    public let description: String?
    public let location: String?
    public let url: URL
}
```

```swift
// Use-case orchestrator — injects store + time for testability
public final class LocalFeedLoader {
    private let store: FeedStore
    private let currentDate: () -> Date

    public init(store: FeedStore, currentDate: @escaping () -> Date) {
        self.store = store
        self.currentDate = currentDate
    }
}

extension LocalFeedLoader: FeedCache {   // conformance exists in the source; nothing consumes it
    public func save(_ feed: [FeedImage]) throws {
        try store.deleteCachedFeed()
        try store.insert(feed.toLocal(), timestamp: currentDate())
    }
}

extension LocalFeedLoader {
    public func load() throws -> [FeedImage] {
        if let cache = try store.retrieve(),
           FeedCachePolicy.validate(cache.timestamp, against: currentDate()) {
            return cache.feed.toModels()
        }
        return []
    }
}

// CQS: `load` is a side-effect-free QUERY. Deleting an invalid/expired cache is a
// separate COMMAND — `validateCache` — run from `sceneWillResignActive`, never inside load.
extension LocalFeedLoader {
    private struct InvalidCache: Error {}

    public func validateCache() throws {
        do {
            if let cache = try store.retrieve(),
               !FeedCachePolicy.validate(cache.timestamp, against: currentDate()) {
                throw InvalidCache()
            }
        } catch {
            try store.deleteCachedFeed()    // delete on expiration OR retrieval error
        }
    }
}
```

> The case study first shipped deletion *inside* `load`, then reverted it when Load and Validate were split into separate use cases. A query that mutates state is the smell; the fix was a requirements change first, code second.

```swift
// Cache policy — encapsulated business rule, no instance state
final class FeedCachePolicy {
    private init() {}

    private static let calendar = Calendar(identifier: .gregorian)
    private static var maxCacheAgeInDays: Int { 7 }

    static func validate(_ timestamp: Date, against date: Date) -> Bool {
        guard let maxCacheAge = calendar.date(byAdding: .day,
                                               value: maxCacheAgeInDays,
                                               to: timestamp) else { return false }
        return date < maxCacheAge
    }
}
```

**Rules**: `LocalFeedLoader` converts between `FeedImage` <-> `LocalFeedImage` at the boundary (private `toLocal()`/`toModels()` `Array` extensions). `currentDate` is injected as a closure for test control. Cache policy is a separate type — not embedded in the loader. Store protocol uses `throws` (not callbacks).

### Why `LocalFeedImage` is not duplication

`LocalFeedImage` mirrors `FeedImage` field for field, and that is the point. They are **Data
Transfer Model Representations**: separate types that happen to look alike today and will change
for different reasons tomorrow — a cache migration must not ripple into the domain, and a domain
addition must not force a store rewrite. Separating them lets each change at its own pace, and
lets the modules be developed in parallel.

The forcing constraint is that **a type can conform to a protocol only once**. A shared model
cannot carry two different `Decodable` implementations for two different APIs, or a `Codable` for
the file system *and* an `@Model` for SwiftData. Whoever gets there first wins permanently.

> **Guardrail**: do not add `Codable`/`@Model`/persistence conformances to `LocalFeedImage` — that is a framework-specific detail. Wrap it in a private type inside the store (`ManagedFeedImage`, `CodableFeedImage`) and convert at the boundary, mirroring the API layer's private-`Decodable` rule.

Beyond correctness, this is about the message left behind: a persistence conformance on a
framework-agnostic model tells the next developer it is fine to add Realm or CoreData details
there too. The mapping cost is a couple of private array extensions — measure before deciding it
is too expensive, and if you have no measurement, optimize for maintainability.

---

## Presentation Layer (Foundation only)

Generic presenter drives loading/error/success state. View protocols use associated types and `@MainActor`. View models are plain structs.

```swift
// View protocols — @MainActor for thread-safe UI updates
@MainActor
public protocol ResourceView {
    associatedtype ResourceViewModel
    func display(_ viewModel: ResourceViewModel)
}

@MainActor
public protocol ResourceLoadingView {
    func display(_ viewModel: ResourceLoadingViewModel)
}

@MainActor
public protocol ResourceErrorView {
    func display(_ viewModel: ResourceErrorViewModel)
}
```

`@MainActor` on view protocols is a **concurrency annotation**, not a framework dependency. The Presentation layer still imports only Foundation. This annotation ensures all display calls happen on the main thread.

```swift
// View models — plain structs
public struct ResourceLoadingViewModel {
    public let isLoading: Bool
}

public struct ResourceErrorViewModel {
    public let message: String?

    static var noError: ResourceErrorViewModel { .init(message: nil) }
    static func error(message: String) -> ResourceErrorViewModel { .init(message: message) }
}
```

```swift
// Generic presenter — reusable across any resource type
@MainActor
public final class LoadResourcePresenter<Resource, View: ResourceView> {
    public typealias Mapper = (Resource) throws -> View.ResourceViewModel

    private let resourceView: View
    private let loadingView: ResourceLoadingView
    private let errorView: ResourceErrorView
    private let mapper: Mapper

    public static var loadError: String {
        NSLocalizedString("GENERIC_CONNECTION_ERROR",
                          tableName: "Shared",
                          bundle: Bundle.module,
                          comment: "Error message displayed when we can't load the resource from the server")
    }

    public init(resourceView: View, loadingView: ResourceLoadingView,
                errorView: ResourceErrorView, mapper: @escaping Mapper) {
        self.resourceView = resourceView
        self.loadingView = loadingView
        self.errorView = errorView
        self.mapper = mapper
    }

    // Convenience: when Resource == View.ResourceViewModel, mapper is identity
    public init(resourceView: View, loadingView: ResourceLoadingView,
                errorView: ResourceErrorView) where Resource == View.ResourceViewModel {
        self.resourceView = resourceView
        self.loadingView = loadingView
        self.errorView = errorView
        self.mapper = { $0 }
    }

    public func didStartLoading() {
        errorView.display(.noError)
        loadingView.display(ResourceLoadingViewModel(isLoading: true))
    }

    public func didFinishLoading(with resource: Resource) {
        do {
            resourceView.display(try mapper(resource))
            loadingView.display(ResourceLoadingViewModel(isLoading: false))
        } catch {
            didFinishLoading(with: error)
        }
    }

    public func didFinishLoading(with error: Error) {
        errorView.display(.error(message: Self.loadError))
        loadingView.display(ResourceLoadingViewModel(isLoading: false))
    }
}
```

**Rules**: Presenter has no UIKit/SwiftUI imports. The `Mapper` typealias allows resource transformation (e.g., `Data -> UIImage`). Identity mapper convenience init eliminates boilerplate when types match.

- **Bundle**: error strings use `NSLocalizedString` with `Bundle.module` in **SPM** targets (this skill's recommended layout — see `spm-project-structure.md`); use `Bundle(for: Self.self)` in a plain **framework** target, where `Bundle.module` does not exist.
- **Presenters create ALL presentation data**, including the screen `title` (a presenter property using `NSLocalizedString`); composers assign it to the view — the view controller stays agnostic of presenters.
- **Locale/time-dependent view models** inject their context as *defaulted* parameters of the static `map`, so production call-sites stay clean and tests pin a fixed locale:

```swift
public static func map(_ comments: [ImageComment],
                       currentDate: Date = Date(),
                       calendar: Calendar = .current,
                       locale: Locale = .current) -> ImageCommentsViewModel { ... }
```

---

## UI Layer — UIKit

`ListViewController` uses `DiffableDataSource`. `CellController` (a struct wrapping any `UITableViewDataSource`, with optional delegate/prefetching discovered via `as?`) type-erases cell behavior — full definition and the "why a struct, not a protocol" rationale live in `adapters-and-proxies.md`. Views conform to presenter protocols.

```swift
// ListViewController — DiffableDataSource + refresh + error view
public final class ListViewController: UITableViewController,
    UITableViewDataSourcePrefetching, ResourceLoadingView, ResourceErrorView {

    private(set) public var errorView = ErrorView()
    public var onRefresh: (() -> Void)?

    public func display(_ sections: [CellController]...) {
        var snapshot = NSDiffableDataSourceSnapshot<Int, CellController>()
        sections.enumerated().forEach { section, cellControllers in
            snapshot.appendSections([section])
            snapshot.appendItems(cellControllers, toSection: section)
        }
        // Animated diff: only changed items are updated. (applySnapshotUsingReloadData was
        // replaced by apply in commit 0fbdd1a.)
        dataSource.apply(snapshot)
    }

    // Diffable only reloads CHANGED items, so it won't reflow on a Dynamic Type change —
    // reload when the content size category changes. `traitCollectionDidChange(_:)` is
    // deprecated since iOS 17; register for the specific trait instead (unregistration
    // happens automatically at the end of the view controller's lifecycle).
    public override func viewDidLoad() {
        super.viewDidLoad()

        registerForTraitChanges([UITraitPreferredContentSizeCategory.self]) { (self: Self, previous: UITraitCollection) in
            self.tableView.reloadData()
        }
    }
}
```

**Rules**: View controllers are thin — display logic only. `onRefresh` closure decouples from specific loading logic. Multiple sections supported via variadic parameter.

> On iOS 17+ there is also automatic trait tracking: traits read inside `layoutSubviews()`,
> `draw(_:)`, `updateConstraints()` and similar update callbacks auto-invalidate when the trait
> changes, with no registration at all — registration is only needed outside those callbacks, as
> here.

---

## UI Layer — SwiftUI Equivalent

Same architectural boundaries, different binding mechanism. An `@Observable` view model replaces the adapter → `WeakRefVirtualProxy` → presenter chain: SwiftUI manages view lifecycle, so the proxy is unnecessary. The Composition Root still creates the view model (injecting the `() async throws -> Resource` loader) and passes it via `@State`/`.environment()`. The presenter's static mappers and localized strings are reused as-is from the Presentation layer.

Deep coverage of the SwiftUI side — `@main App` composition root, environment DI, routing at the
root, `@Observable` presenters — now lives in [swiftui-composition.md](swiftui-composition.md).
For SwiftUI specifics (state management, view composition, `@Observable` patterns), use the `swiftui-expert` skill.

---

## Guardrails

- Do not import UIKit/SwiftUI in Feature, API, Cache, or Presentation layers — Foundation only
- Do not use `class` for domain models — use `struct` with `Sendable` and `Equatable` (`Hashable` when it is used as an identity or diffable key)
- Do not embed cache policy in the loader — keep it as a separate type with `static` methods
- Do not leak `Decodable` types beyond the mapper — keep them `private`
- Do not add `@MainActor` to domain models or store protocols — isolation is decided at the composition level

## Verification

- [ ] Feature layer has zero framework imports beyond Foundation
- [ ] All domain models are `struct`, `Sendable`, and `Equatable` or `Hashable`
- [ ] No layer references a concrete type from another layer — every boundary is a function type (single method) or a protocol (several cohesive methods, a named contract, or meaningful conformance)
- [ ] View protocols (`ResourceView`, `ResourceLoadingView`, `ResourceErrorView`) are `@MainActor`
- [ ] Mapper Decodable types are `private`
- [ ] `LocalFeedImage` is decoupled from `FeedImage` (separate type)
- [ ] Cache policy has no instance state — `static` methods only
