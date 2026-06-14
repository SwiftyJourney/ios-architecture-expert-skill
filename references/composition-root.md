# Composition Root — Code Reference

Use this when:
- You need to wire dependencies in the Composition Root or `FeedService`
- You are creating a new `UIComposer` static factory
- You need the Scheduler protocol for CoreData/InMemory polymorphism
- You need the fallback strategy (CoreData -> InMemory)
- You need to understand SceneDelegate wiring or cache validation

Skip this file if:
- You need adapter/proxy/pagination wiring details. Use `adapters-and-proxies.md`.
- You need concurrency rationale (`@Sendable`, `Task.immediate`). Use `concurrency-at-boundaries.md`.
- You need to understand architectural layers. Use `architecture-layers.md`.

Jump to:
- [FeedService — @MainActor Orchestrator](#feedservice--mainactor-orchestrator)
- [Scheduler Protocol](#scheduler-protocol--abstract-execution-context)
- [InMemoryFeedStore — Fallback](#inmemoryFeedstore--mainactor-fallback)
- [CoreDataFeedStore — Sendable Wrapper](#coredatafeedstore--sendable-wrapper)
- [SceneDelegate Wiring](#scenedelegate-wiring)
- [Debug-only code: the `#if DEBUG` subclass pattern](#debug-only-code-the-if-debug-subclass-pattern)
- [UI Composer — UIKit](#ui-composer--uikit-feeduicomposer)
- [UI Composer — SwiftUI](#ui-composer--swiftui-equivalent)

---

## FeedService — @MainActor Orchestrator

The Composition Root is the **only place** that knows about all modules. It creates concrete instances, wires dependencies, and defines fallback strategies.

Composers, presentation adapters, and proxies belong to the **app target**, never the UI framework module — the case study physically moved them out of the iOS framework once the app target existed. Note the deliberate composition asymmetry: feed loading is **remote-first** (freshness), image loading is **local-first** (image data per URL is immutable, so cache hits are always valid).

```swift
@MainActor
final class FeedService {
    // Lazy init — defer expensive creation until first use
    private lazy var httpClient: HTTPClient = {
        URLSessionHTTPClient(session: URLSession(configuration: .ephemeral))
    }()

    private lazy var logger = Logger(subsystem: "com.example.FeedApp", category: "main")

    private lazy var store: FeedStore & FeedImageDataStore & Scheduler & Sendable = {
        do {
            return try CoreDataFeedStore(
                storeURL: NSPersistentContainer
                    .defaultDirectoryURL()
                    .appendingPathComponent("feed-store.sqlite"))
        } catch {
            assertionFailure("Failed to instantiate CoreData store with error: \(error.localizedDescription)")
            logger.fault("Failed to instantiate CoreData store with error: \(error.localizedDescription)")
            return InMemoryFeedStore()  // Fallback — graceful degradation
        }
    }()

    private lazy var baseURL = URL(string: "https://api.example.com/feed")!

    // Convenience init for testing — inject all dependencies
    convenience init(httpClient: HTTPClient,
                     store: FeedStore & FeedImageDataStore & Scheduler & Sendable) {
        self.init()
        self.httpClient = httpClient
        self.store = store
    }
}
```

**Key principles**:
- `lazy` properties defer creation and allow override in tests via `convenience init`
- Protocol composition (`FeedStore & FeedImageDataStore & Scheduler & Sendable`) constrains the store type at the wiring site
- Fallback pattern: CoreData -> InMemory on failure
- `@MainActor` ensures all composition happens on the main thread

### Remote-with-local-fallback pattern

```swift
func loadRemoteFeedWithLocalFallback() async throws -> Paginated<FeedImage> {
    do {
        let feed = try await loadAndCacheRemoteFeed()
        return makeFirstPage(items: feed)
    } catch {
        let feed = try await loadLocalFeed()
        return makeFirstPage(items: feed)
    }
}

private func loadAndCacheRemoteFeed() async throws -> [FeedImage] {
    let feed = try await loadRemoteFeed()
    await store.schedule { [store] in
        let loader = LocalFeedLoader(store: store, currentDate: Date.init)
        try? loader.save(feed)
    }
    return feed
}

private func loadLocalFeed() async throws -> [FeedImage] {
    try await store.schedule { [store] in
        let localFeedLoader = LocalFeedLoader(store: store, currentDate: Date.init)
        return try localFeedLoader.load()
    }
}

private func loadRemoteFeed(after: FeedImage? = nil) async throws -> [FeedImage] {
    let url = FeedEndpoint.get(after: after).url(baseURL: baseURL)
    let (data, response) = try await httpClient.get(from: url)
    return try FeedItemsMapper.map(data, from: response)
}
```

`try? loader.save(feed)` is fire-and-forget by design — the customer still sees the feed if caching fails. Cache-on-success is a Composition Root concern: historically an explicit Decorator with a narrow save-only `FeedCache` protocol owned by the feature module.

### Parallel loading with `async let`

Zipping the cache load with the remote page exists so accumulated items are **not** held in closure memory — the cache is the single source of truth for previous pages. Reduced latency is a side benefit, not the motivation:

```swift
private func loadMoreRemoteFeed(last: FeedImage?) async throws -> Paginated<FeedImage> {
    async let cachedItems = try await loadLocalFeed()
    async let newItems = try await loadRemoteFeed(after: last)

    let items = try await cachedItems + newItems

    await store.schedule { [store] in
        let localFeedLoader = LocalFeedLoader(store: store, currentDate: Date.init)
        try? localFeedLoader.save(items)
    }

    return try await makePage(items: items, last: newItems.last)
}
```

### Recursive pagination composition

```swift
private func makeFirstPage(items: [FeedImage]) -> Paginated<FeedImage> {
    makePage(items: items, last: items.last)
}

private func makePage(items: [FeedImage], last: FeedImage?) -> Paginated<FeedImage> {
    Paginated(items: items, loadMore: last.map { last in
        { @MainActor @Sendable in try await self.loadMoreRemoteFeed(last: last) }
    })
}
```

The `last` item determines if there are more pages. When `last` is `nil`, `loadMore` is `nil` — pagination ends. The closure captures `self` (the `FeedService`) to enable recursive page loading. Both `@MainActor` and `@Sendable` annotations are required at this composition boundary.

### Cache validation (fire-and-forget)

```swift
func validateCache() {
    Task.immediate { @MainActor in
        await store.schedule { [store, logger] in
            do {
                let localFeedLoader = LocalFeedLoader(store: store, currentDate: Date.init)
                try localFeedLoader.validateCache()
            } catch {
                logger.error("Failed to validate cache with error: \(error.localizedDescription)")
            }
        }
    }
}
```

`Task.immediate` runs synchronously up to the first suspension point. This ensures the cache validation starts immediately when the scene resigns active. Errors are logged but do not propagate — this is intentional fire-and-forget.

### Image loading with local-first strategy

```swift
func loadLocalImageWithRemoteFallback(url: URL) async throws -> Data {
    do {
        return try await loadLocalImage(url: url)
    } catch {
        return try await loadAndCacheRemoteImage(url: url)
    }
}

private func loadLocalImage(url: URL) async throws -> Data {
    try await store.schedule { [store] in
        let localImageLoader = LocalFeedImageDataLoader(store: store)
        return try localImageLoader.loadImageData(from: url)
    }
}

private func loadAndCacheRemoteImage(url: URL) async throws -> Data {
    let (data, response) = try await httpClient.get(from: url)
    let imageData = try FeedImageDataMapper.map(data, from: response)
    await store.schedule { [store] in
        let localImageLoader = LocalFeedImageDataLoader(store: store)
        try? localImageLoader.save(data, for: url)
    }
    return imageData
}
```

### Comments loading (closure factory)

```swift
func loadComments(for image: FeedImage) -> () async throws -> [ImageComment] {
    return { [httpClient, baseURL] in
        let url = ImageCommentsEndpoint.get(image.id).url(baseURL: baseURL)
        let (data, response) = try await httpClient.get(from: url)
        return try ImageCommentsMapper.map(data, from: response)
    }
}
```

Returns a closure — not the result. The Composition Root creates the closure, the adapter calls it when the user navigates. This defers execution and keeps navigation decoupled from loading.

---

## Scheduler Protocol — Abstract Execution Context

The Scheduler protocol abstracts whether store operations run on CoreData's background context or InMemory's main actor. This enables polymorphism without leaking threading details into the service layer. The protocol deliberately lives in the Composition Root (`FeedService.swift`), not the feature module — it is a wiring concern.

```swift
protocol Scheduler {
    @MainActor
    func schedule<T>(_ action: @escaping @Sendable () throws -> T) async rethrows -> T
}

// CoreData: dispatch to context queue
extension CoreDataFeedStore: Scheduler {
    @MainActor
    func schedule<T>(_ action: @escaping @Sendable () throws -> T) async rethrows -> T {
        if contextQueue == .main { return try action() }
        else { return try await perform(action) }
    }
}

// InMemory: execute immediately on main
extension InMemoryFeedStore: Scheduler {
    @MainActor
    func schedule<T>(_ action: @escaping @Sendable () throws -> T) async rethrows -> T {
        try action()
    }
}
```

**Why this abstraction exists**: The service code must not know whether it is using CoreData or InMemory. Both satisfy `FeedStore & FeedImageDataStore & Scheduler & Sendable`. Production uses `CoreDataFeedStore(.background)` and gets background-context dispatch. Acceptance tests inject the **real** `CoreDataFeedStore` with `contextQueue: .main` and get synchronous execution via the `.main` short-circuit above.

---

## InMemoryFeedStore — @MainActor Fallback

**Production-only** graceful-degradation fallback when CoreData initialization fails — not a test double. NSCache-backed image storage gives automatic eviction under memory pressure. It is contract-tested via the same `FeedStoreSpecs`/`FeedImageDataStoreSpecs` shared specs as CoreData.

```swift
@MainActor
public class InMemoryFeedStore {
    private var feedCache: CachedFeed?
    private var feedImageDataCache = NSCache<NSURL, NSData>()

    public init() {}
}

extension InMemoryFeedStore: FeedStore {
    public func deleteCachedFeed() throws {
        feedCache = nil
    }

    public func insert(_ feed: [LocalFeedImage], timestamp: Date) throws {
        feedCache = CachedFeed(feed: feed, timestamp: timestamp)
    }

    public func retrieve() throws -> CachedFeed? {
        feedCache
    }
}

extension InMemoryFeedStore: FeedImageDataStore {
    public func insert(_ data: Data, for url: URL) throws {
        feedImageDataCache.setObject(data as NSData, forKey: url as NSURL)
    }

    public func retrieve(dataForURL url: URL) throws -> Data? {
        feedImageDataCache.object(forKey: url as NSURL) as Data?
    }
}
```

`@MainActor` is appropriate here because all composition code runs on the main actor. Acceptance tests do **not** use this store — they inject the real `CoreDataFeedStore(storeURL: URL(fileURLWithPath: "/dev/null"), contextQueue: .main)`, which is effectively in-memory and synchronous. Test cache states (`.empty` / `.withExpiredFeedCache` / `.withNonExpiredFeedCache`) are `CoreDataFeedStore` factory extensions in the test target.

---

## CoreDataFeedStore — Sendable Wrapper

The CoreData store is `Sendable` and wraps all context operations through a single async `perform<T>` method.

```swift
public final class CoreDataFeedStore: Sendable {
    private let container: NSPersistentContainer
    let context: NSManagedObjectContext

    public enum ContextQueue {
        case main
        case background
    }

    public var contextQueue: ContextQueue {
        context == container.viewContext ? .main : .background
    }

    @MainActor
    public convenience init(storeURL: URL, contextQueue: ContextQueue = .background) throws {
        guard let model = CoreDataFeedStore.model else {
            throw StoreError.modelNotFound
        }
        try self.init(storeURL: storeURL, contextQueue: contextQueue, model: model)
    }

    public func perform<T>(_ action: @escaping @Sendable () throws -> T) async rethrows -> T {
        try await context.perform(action)
    }

    deinit {
        cleanUpReferencesToPersistentStores()
    }
}
```

**Key details**:
- `Sendable` conformance enables safe passing across actor boundaries
- `contextQueue` computed property lets `Scheduler` decide between synchronous (main) and async (background) execution
- `@MainActor` on the convenience init ensures the shared `model` is accessed safely
- The static `model` is loaded **once** and cached — loading the same `NSManagedObjectModel` twice registers duplicate `NSEntityDescription`s claiming the same `NSManagedObject` subclasses (undefined behavior)
- `perform<T>` delegates to `NSManagedObjectContext.perform` for thread-safe CoreData access
- `deinit` cleans up persistent store references — the CoreData stack lives and dies with the instance, which matters when tests create many stores

---

## SceneDelegate Wiring

```swift
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?

    private lazy var feedService = FeedService()

    private lazy var navigationController = UINavigationController(
        rootViewController: FeedUIComposer.feedComposedWith(
            feedLoader: feedService.loadRemoteFeedWithLocalFallback,
            imageLoader: feedService.loadLocalImageWithRemoteFallback,
            selection: showComments))

    // Test injection — override feedService with custom dependencies
    convenience init(httpClient: HTTPClient, store: FeedStore & FeedImageDataStore & Scheduler & Sendable) {
        self.init()
        self.feedService = FeedService(httpClient: httpClient, store: store)
    }

    func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options connectionOptions: UIScene.ConnectionOptions) {
        guard let scene = (scene as? UIWindowScene) else { return }
        window = UIWindow(windowScene: scene)
        configureWindow()
    }

    func configureWindow() {
        window?.rootViewController = navigationController
        window?.makeKeyAndVisible()
    }

    func sceneWillResignActive(_ scene: UIScene) {
        feedService.validateCache()
    }

    private func showComments(for image: FeedImage) {
        let comments = CommentsUIComposer.commentsComposedWith(commentsLoader: feedService.loadComments(for: image))
        navigationController.pushViewController(comments, animated: true)
    }
}
```

**Key techniques**:
- `convenience init` accepts protocol-composed dependencies for acceptance test injection
- `sceneWillResignActive` triggers cache validation (fire-and-forget via `Task.immediate`)
- Navigation closures (`showComments`) keep view controllers decoupled from loading logic
- `lazy` properties ensure composition happens once, on first access

### Debug-only code: the `#if DEBUG` subclass pattern

When you legitimately need debug/test-only behavior in the composition root — a debug menu, a simulator-only seam, canned responses for UI tests — **do not scatter `#if DEBUG` branches through the production `SceneDelegate`.** Extract it into a `#if DEBUG`-guarded *subclass* that overrides the production composition's factory seams, and select that subclass only in debug builds. Release builds never compile, reference, or ship the debug code.

```swift
// AppDelegate — pick the debug delegate only in DEBUG builds
func application(_ app: UIApplication,
                 configurationForConnecting session: UISceneSession,
                 options: UIScene.ConnectionOptions) -> UISceneConfiguration {
    let configuration = UISceneConfiguration(name: nil, sessionRole: session.role)
    #if DEBUG
    configuration.delegateClass = DebuggingSceneDelegate.self
    #endif
    return configuration
}

#if DEBUG
final class DebuggingSceneDelegate: SceneDelegate {
    override func scene(_ scene: UIScene, willConnectTo session: UISceneSession,
                        options: UIScene.ConnectionOptions) {
        if CommandLine.arguments.contains("-reset") {
            try? FileManager.default.removeItem(at: localStoreURL)   // clean state per UI-test run
        }
        super.scene(scene, willConnectTo: session, options: options)
    }

    // Override a production factory seam to swap in a canned-response client
    override func makeRemoteClient() -> HTTPClient {
        if let connectivity = UserDefaults.standard.string(forKey: "connectivity") {
            return DebuggingHTTPClient(connectivity: connectivity)   // "online"/"offline" set by the UI test
        }
        return super.makeRemoteClient()
    }
}
#endif
```

**Why**: the production `SceneDelegate` stays free of test/debug concerns (it just exposes overridable factory seams like `makeRemoteClient()`); the `#if DEBUG` guard guarantees none of this is in the release binary; and the launch-argument/`UserDefaults` flags (`-reset`, `connectivity`) let an out-of-process UI test drive app state. The same shape generalizes to any debug-only feature — isolate it in a `#if DEBUG` type, don't litter production with directives.

> For **acceptance tests specifically**, prefer the in-process `convenience init(httpClient:store:)` seam above over launch-argument hooks — it's faster and needs no debug subclass (the case study ultimately deleted its UI-test target in favor of in-process tests; see `testing-strategy.md`). The `#if DEBUG`-isolated-subclass technique remains the correct way to keep *genuinely* debug-only code (debug menus, simulator seams) out of release.

---

## UI Composer — UIKit (`FeedUIComposer`)

Static factory that wires the presenter->adapter->view chain for a feature. Returns a fully configured view controller.

```swift
@MainActor
public final class FeedUIComposer {
    private init() {}  // Static factory — no instances

    public static func feedComposedWith(
        feedLoader: @MainActor @escaping () async throws -> Paginated<FeedImage>,
        imageLoader: @MainActor @escaping (URL) async throws -> Data,
        selection: @MainActor @escaping (FeedImage) -> Void = { _ in }
    ) -> ListViewController {
        let presentationAdapter = LoadResourcePresentationAdapter<Paginated<FeedImage>, FeedViewAdapter>(
            loader: feedLoader
        )

        let feedController = makeFeedViewController(title: FeedPresenter.title)
        feedController.onRefresh = presentationAdapter.loadResource

        presentationAdapter.presenter = LoadResourcePresenter(
            resourceView: FeedViewAdapter(
                controller: feedController,
                imageLoader: imageLoader,
                selection: selection),
            loadingView: WeakRefVirtualProxy(feedController),
            errorView: WeakRefVirtualProxy(feedController))

        return feedController
    }
}
```

**Wiring chain**: `ListViewController.onRefresh` -> `PresentationAdapter.loadResource` -> async loader -> `Presenter.didFinishLoading` -> `FeedViewAdapter.display` -> `ListViewController.display([CellController]...)`

---

## UI Composer — SwiftUI Equivalent

```swift
@MainActor
struct FeedUIComposer {
    static func feedComposedWith(
        feedLoader: @escaping () async throws -> Paginated<FeedImage>,
        imageLoader: @escaping (URL) async throws -> Data,
        selection: @escaping (FeedImage) -> Void = { _ in }
    ) -> some View {
        let viewModel = FeedViewModel(loader: feedLoader, imageLoader: imageLoader)

        FeedView(viewModel: viewModel, selection: selection)
            .task { await viewModel.load() }
    }
}
```

In SwiftUI, `@Observable` replaces the adapter->proxy->presenter chain. The view model directly holds loading/error/success state and SwiftUI observes mutations automatically. No `WeakRefVirtualProxy` needed — SwiftUI manages view lifecycle.

---

## Guardrails

- Do not create concrete types inside feature modules — all instantiation belongs in the Composition Root
- Do not use singletons — `lazy var` in `FeedService` achieves deferred init without global state
- Do not skip the fallback — always handle CoreData initialization failure with `InMemoryFeedStore`
- Do not use `DispatchQueue.main.async` in composition — use `@MainActor` annotation instead
- Do not embed navigation logic in view controllers — use closure callbacks wired in the Composition Root
- Do not scatter `#if DEBUG` branches through the production composition — isolate debug/test-only code in a `#if DEBUG`-guarded `SceneDelegate` subclass selected via `configuration.delegateClass`, so it never ships in release

## Verification

- [ ] `FeedService` is `@MainActor` and uses `lazy var` for all dependencies
- [ ] Store type is constrained as `FeedStore & FeedImageDataStore & Scheduler & Sendable`
- [ ] CoreData initialization failure falls back to `InMemoryFeedStore`
- [ ] `convenience init` exists for test injection on both `FeedService` and `SceneDelegate`
- [ ] `validateCache()` uses `Task.immediate` (not bare `Task`)
- [ ] Pagination `loadMore` closure is annotated `@MainActor @Sendable`
- [ ] All `Scheduler.schedule` calls capture `[store]` (not `[self]`)
