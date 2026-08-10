# SwiftUI Composition

Use this when:
- You are wiring a SwiftUI app and need the `@main App` equivalent of the SceneDelegate composition root
- You are deciding between init injection and the environment for a dependency
- Navigation state needs an owner and views are starting to construct their own destinations
- You need the `@Observable` equivalent of the presenter/adapter wiring
- You need acceptance-test seams for a SwiftUI app

Skip this file if:
- You need SwiftUI view *code* — state, layout, styling, animation. Use the `swiftui-expert` skill.
- You need UIKit composition (`SceneDelegate`, `FeedUIComposer`). Use `composition-root.md`.
- You are embedding SwiftUI in UIKit or vice versa. Use the `swiftui-uikit-interop` skill.

The principles are identical to `composition-root.md` — one place composes everything, modules
stay agnostic of each other, asynchrony and navigation are composition details. This file is the
SwiftUI dialect, not a different architecture.

Jump to:
- [The @main App is the Composition Root](#the-main-app-is-the-composition-root)
- [Init injection vs the environment](#init-injection-vs-the-environment)
- [@Observable view model as the presenter](#observable-view-model-as-the-presenter)
- [Navigation belongs to the root](#navigation-belongs-to-the-root)
- [What replaces the UIKit patterns](#what-replaces-the-uikit-patterns)
- [Acceptance-test seams](#acceptance-test-seams)

---

## The @main App is the Composition Root

`@main App` + `Scene` replaces `AppDelegate` + `SceneDelegate` as the process entry point — and
therefore as the Composition Root. Same three consequences as UIKit: it is the only place that
imports every module, the only place object lifetimes are decided, and nothing depends on it.

```swift
import Feed
import FeedAPI
import FeedCache
import FeedCacheInfra
import FeedPresentation
import FeedSwiftUI

@main
struct FeedApp: App {
    // The root owns the graph. @State gives struct-App storage that survives re-instantiation.
    @State private var composer = AppComposer()

    var body: some Scene {
        WindowGroup {
            composer.makeRootView()
        }
    }
}

@MainActor
final class AppComposer {
    // A plain class, deliberately NOT @Observable: no view observes the composer, and
    // @Observable rejects `lazy` stored properties anyway. @State just holds it.

    // Infrastructure — created once, lazily, exactly like the SceneDelegate's lazy vars
    private lazy var httpClient: HTTPClient = URLSessionHTTPClient(
        session: URLSession(configuration: .ephemeral)
    )

    private static let storeURL = URL.applicationSupportDirectory.appending(path: "feed-store.sqlite")

    private lazy var store: FeedStore & FeedImageDataStore = {
        do {
            return try CoreDataFeedStore(storeURL: Self.storeURL)
        } catch {
            // in-memory fallback vs Null Object is a product decision — log either way
            return InMemoryFeedStore()
        }
    }()

    private lazy var feedService = FeedService(store: store, httpClient: httpClient)

    private var navigation = NavigationModel()

    func makeRootView() -> some View {
        NavigationRootView(navigation: navigation, composer: self)
    }

    func makeFeedView() -> some View {
        let model = FeedViewModel(
            loadFeed: feedService.loadFeed,
            onSelect: { [navigation] image in navigation.path.append(.comments(for: image)) }
        )
        return FeedView(model: model)
    }

    func makeCommentsView(for image: FeedImage) -> some View {
        CommentsView(model: CommentsViewModel(
            loadComments: feedService.makeCommentsLoader(for: image)
        ))
    }
}
```

What did *not* change from `composition-root.md`: `FeedService` (the `@MainActor` orchestrator),
the store fallback decision, the remote-with-local-fallback strategies, logging from the root.
The UI dialect changed; the object graph did not — `FeedService` and everything behind it is
shared verbatim between a UIKit and a SwiftUI app. That is the payoff of keeping composition out
of the modules.

**App re-instantiation gotcha:** the `App` struct can be re-created by the system; plain `let`
properties would rebuild the graph. `@State` (or a `static let`) is what guarantees
compose-exactly-once.

---

## Init injection vs the environment

Two mechanisms exist; they are not interchangeable:

| | Init injection | Environment (`@Environment` / `.environment`) |
|---|---|---|
| Missing dependency | **Compile error** | Runtime crash (observable objects) or silent default (values) |
| Visibility | Explicit in the initializer | Ambient — invisible at the call site |
| Best for | The view's *required* collaborators — view models, loaders | Cross-cutting values many views read — theme, feature flags, the navigation model |
| Analogue | Constructor injection | Ambient Context (see `dependency-management.md` — with the same caveats) |

Rules that keep it honest:

- **Default to init injection.** A `FeedView` that cannot exist without a `FeedViewModel` should
  say so in its initializer, and the composer provides it. This is the constrained-construction
  lesson from `dependency-management.md` working *for* you: SwiftUI views are structs you
  construct directly, so the compiler enforces the contract for free.
- **The environment is set once, at the root, by the composer** — `.environment(navigation)` on
  the root view. A feature view reading `@Environment(NavigationModel.self)` is coupling to an
  ambient contract; keep that to genuinely cross-cutting concerns, and never *write* app state
  through the environment from a leaf.
- **`@Entry` declares custom keys without the `EnvironmentKey` boilerplate:**

```swift
extension EnvironmentValues {
    @Entry var imageCache: ImageCache = .null   // Null Object default, not a crash
}
```

- Give observable environment dependencies a **Null Object default** where a missing value has a
  sane neutral behaviour — a crash-on-missing default is a runtime landmine that init injection
  would have caught at compile time.
- Modules never read the environment for *domain* dependencies. If `FeedSwiftUI` needs a loader,
  it arrives through an initializer, composed at the root — the environment is not a service
  locator.

---

## @Observable view model as the presenter

The `LoadResourcePresenter` job — turn resource events into view state — becomes an `@Observable`
class whose stored properties *are* the view state. The adapter job (own the task, drive the
presenter) folds into the same type, because observation replaces the view-protocol indirection:

```swift
// FeedSwiftUI target — platform-agnostic apart from SwiftUI itself
import Observation

@MainActor
@Observable
public final class FeedViewModel {
    public private(set) var items: [FeedImage] = []
    public private(set) var isLoading = false
    public private(set) var errorMessage: String?

    private let loadFeed: () async throws -> Paginated<FeedImage>
    private let onSelect: (FeedImage) -> Void
    private var loadTask: Task<Void, Never>?

    public init(loadFeed: @escaping () async throws -> Paginated<FeedImage>,
                onSelect: @escaping (FeedImage) -> Void) {
        self.loadFeed = loadFeed
        self.onSelect = onSelect
    }

    public func load() async {
        isLoading = true
        errorMessage = nil
        defer { isLoading = false }
        do {
            items = try await loadFeed().items
        } catch {
            errorMessage = FeedPresenter.loadError   // strings still come from Presentation
        }
    }

    public func select(_ item: FeedImage) { onSelect(item) }
}

public struct FeedView: View {
    @Bindable var model: FeedViewModel

    public var body: some View {
        List(model.items) { item in
            FeedImageCard(item: item)
                .onTapGesture { model.select(item) }
        }
        .refreshable { await model.load() }
        .task { await model.load() }
    }
}
```

Load-bearing details:

- **The closure is still the boundary.** `loadFeed: () async throws -> Paginated<FeedImage>` is
  injected by the composer — the view model has no idea whether it is remote, cached, composite,
  or paginated. Identical rule to the UIKit adapters.
- **Presentation strings stay in `FeedPresentation`.** The view model maps errors to
  `FeedPresenter`-provided strings; it does not invent user-facing copy. The cross-platform
  presentation layer survives intact — this class is the *binder*, replacing the view-controller
  binder from MVVM-in-UIKit (`presentation-patterns.md`).
- **`.task` gives structured cancellation for free** — the task cancels when the view disappears,
  which replaces the `didEndDisplaying` cancellation choreography. Work that must *outlive* the
  view (fire-and-forget cache validation) does not belong in the view model — it belongs in
  `FeedService`, behind the composed closure.
- **Ownership:** the composer creates the view model; the *owning view* holds it with `@State`
  (or the composer holds long-lived ones, as `AppComposer` does). Child views receive it as a
  plain parameter or `@Bindable`. Two views needing the same instance is a composition decision —
  made at the root, not by a view reaching into the environment.
- Testing these view models: `@MainActor` suites, spy-driven completion, await observable state —
  worked through in the swift-testing-expert skill (`isolation-and-sendability.md`).

---

## Navigation belongs to the root

The coordinator question ("who owns navigation?") has a crisp SwiftUI answer: **a navigation model
owned by the Composition Root**, rendered by `NavigationStack(path:)`. Feature modules never
navigate — they emit intents.

```swift
// App target — routes are a composition concern, so the enum lives at the root
enum Route: Hashable {
    case comments(for: FeedImage)
    case imageDetail(FeedImage)
}

@MainActor
@Observable
final class NavigationModel {
    var path: [Route] = []
}

struct NavigationRootView: View {
    @Bindable var navigation: NavigationModel
    let composer: AppComposer

    var body: some View {
        NavigationStack(path: $navigation.path) {
            composer.makeFeedView()
                .navigationDestination(for: Route.self) { route in
                    switch route {
                    case let .comments(image): composer.makeCommentsView(for: image)
                    case let .imageDetail(image): composer.makeImageDetailView(for: image)
                    }
                }
        }
    }
}
```

- **Feature views expose `onSelect`-style closures; the composer decides what selection means**
  (`navigation.path.append(.comments(for: image))`). This is exactly the module-007 lesson —
  navigation is composed in the root, so `Feed` and `ImageComments` never learn about each other.
- **Deep linking and state restoration fall out for free**: a URL maps to a `[Route]`, assigned to
  `path`; `path` is `Codable`-friendly for scene storage. Both are root concerns — no view changes.
- Tabs/split views: same shape, one `NavigationModel` per stack, all owned at the root.
- A `Router` protocol abstraction over this is a one-implementation protocol — the skill's rule 12
  applies. The concrete `NavigationModel` in the app target is enough until a second
  implementation exists.

---

## What replaces the UIKit patterns

The pattern *intent* survives; the mechanics often dissolve into the framework
(see `adapters-and-proxies.md` → "Patterns you no longer hand-roll"):

| UIKit mechanic | SwiftUI replacement | What remains yours |
|---|---|---|
| `FeedUIComposer` | Composer's `makeFeedView()` factories | Everything — still hand-written wiring |
| `LoadResourcePresentationAdapter` | `.task` + `@Observable` view model | Task ownership moves into `.task`/the VM |
| View protocols + `WeakRefVirtualProxy` | Observation tracking | Nothing — value-type views can't retain-cycle the VM |
| `CellController` type erasure | `ForEach` over `Identifiable` items | Heterogeneous lists: `enum` rows or `AnyView`-free `@ViewBuilder` |
| `didEndDisplaying` cancellation | `.task` auto-cancellation | Prefetch/cancel policy, if you need one, stays in the VM |
| Composite / Decorator / fallback strategies | **Unchanged** | They live behind the closures, below the UI — untouched by the UI dialect |

The memory-management chapter shrinks but does not vanish: `@Observable` view models are still
reference types, and closures the *composer* hands them (`onSelect`) can capture other long-lived
objects — the "memory management belongs in the composer" rule from `presentation-patterns.md`
still applies; it is just that the *views* are no longer part of any cycle.

---

## Acceptance-test seams

The UIKit acceptance test launched `SceneDelegate.configureWindow()`. The SwiftUI equivalent
composes the same root with stubbed infrastructure — no `UIApplication`, no scene machinery:

```swift
@MainActor
@Suite struct FeedAcceptanceTests {
    @Test func onLaunch_displaysRemoteFeedWhenCustomerHasConnectivity() async throws {
        let composer = AppComposer(httpClient: HTTPClientStub.online(response),
                                   store: InMemoryFeedStore())

        let feed = composer.makeFeedViewModel()      // the composer exposes the model it wires
        await feed.load()

        #expect(feed.items.count == 2)
    }
}
```

Make the composer's infrastructure **injectable with production defaults**
(`init(httpClient: HTTPClient = ..., store: FeedStore & FeedImageDataStore = ...)`) — the same
constrained-construction seam the UIKit root uses. Replace only infrastructure, never business
logic; assert on view-model state, which *is* the rendered state under observation. Pixel-level
verification stays with snapshot tests (host the view in a `UIHostingController`); wiring
verification lives here, fast and in-process.

---

## Guardrails

- Do not compose the object graph anywhere but the `@main App`/root composer — a view constructing its own view model is the massive-VC seed in SwiftUI form
- Do not pass domain dependencies through the environment — init injection at the root; environment is for cross-cutting values
- Do not let feature views append to a navigation path or construct destination views — they emit closures; the root routes
- Do not put user-facing strings in view models or views — they come from the Presentation layer
- Do not hold a `Task` in a view model for work that must survive the view — that work belongs in the service layer behind the closure
- Do not create a `Router`/`Coordinator` protocol with one implementation — the concrete navigation model in the app target is the abstraction
- Do not rebuild the graph on `App` re-instantiation — own it with `@State` or `static let`

## Verification

- [ ] All module imports converge only in the app target/composer
- [ ] Every view model is created by the composer; views receive them, never build them
- [ ] Environment carries only cross-cutting values, each with a sane (Null Object) default
- [ ] Navigation path and `navigationDestination` mapping live at the root; features emit intents
- [ ] Presentation strings come from the Presentation module (localized tables intact)
- [ ] The composer's infrastructure is injectable — acceptance tests run the real graph with stubbed HTTP/store
- [ ] `FeedService` and everything below it is untouched by the choice of UI dialect
