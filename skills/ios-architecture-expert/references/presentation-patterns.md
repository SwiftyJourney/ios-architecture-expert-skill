# Presentation Patterns — MVC, MVVM, MVP

Use this when:
- You are choosing between MVC, MVVM and MVP — or asked "which one and why?"
- A view controller has grown unmanageable and you need to know what to extract
- You need presentation logic reused across platforms (iOS + watchOS/macOS) or across features
- You need to understand *why* `WeakRefVirtualProxy` and the presentation adapter exist
- You are moving from a feature-specific presenter to the generic `LoadResourcePresenter`

Skip this file if:
- You need the presenter/view-model *code*. Use `architecture-layers.md`.
- You need adapter/proxy/composite wiring code. Use `adapters-and-proxies.md`.
- You need where instances get created. Use `composition-root.md`.
- You need SwiftUI view code specifics. Use the `swiftui-expert` skill.

Jump to:
- [The one-sentence differences](#the-one-sentence-differences)
- [MVC — many tiny MVCs](#mvc--many-tiny-mvcs)
- [Diagnosing an unsustainable controller](#diagnosing-an-unsustainable-controller)
- [MVVM](#mvvm)
- [MVP](#mvp)
- [Memory management belongs in the composer](#memory-management-belongs-in-the-composer)
- [Choosing — and why "none of them" is the honest answer](#choosing--and-why-none-of-them-is-the-honest-answer)
- [Where presentation strings are created](#where-presentation-strings-are-created)
- [The generalization arc](#the-generalization-arc-feature-specific--generic)

---

## The one-sentence differences

All three are the same idea — separate what is displayed from how it is displayed — differing
only in **who holds a reference to whom**.

| | Holds a reference to the view? | View knows it? | View reference type |
|---|---|---|---|
| **MVC** | Controller → view | No | **Concrete** view type |
| **MVVM** | Nobody — communication is a *binding* | Via binding | — |
| **MVP** | Presenter → view | Yes, two-way | **Abstract** (protocol) |

Everything else — testability, cross-platform reuse, retain cycles — follows from that row.

---

## MVC — many tiny MVCs

> **The biggest misconception with MVC is that there is one model, one view and one controller
> per screen.** A screen is composed of *many* small MVCs.

A list screen is not one MVC. It is:

- one MVC for the list itself (model: the array; view: the table; controller: the list controller)
- one MVC **per cell** (model: one item; view: the cell; controller: a cell controller)
- one MVC for the refresh control, one for the error view, one for the load-more cell…

The controller is a **mediator**. Views do not depend on controllers or models; models do not
depend on controllers or views. Only the controller knows both sides.

**Start with MVC.** It is the simplest solution that works, and refactoring MVC → MVVM → MVP is
mechanical once the responsibilities are already separated. Reaching for a heavier pattern before
you have the problem it solves is the more expensive mistake.

### Extracting a tiny MVC

Every extraction moves responsibilities out but hands the original type **two new ones**: creating
the collaborator, and communicating with it. Watch for that — it is why extraction alone does not
fix a bloated controller, and why the creation half belongs in the Composition Root.

```swift
// Before: the list controller owns the refresh control, its target-action, and the loading call.
// After: a controller per concern, with a closure as the output boundary.
final class RefreshController: NSObject {
    private(set) lazy var view = binded(UIRefreshControl())
    var onRefresh: (() -> Void)?          // output boundary — it does not know what happens next

    @objc func refresh() { onRefresh?() }
}
```

The closure is the point: the extracted controller announces *what happened*, never decides *what
to do about it*. That decision belongs to whoever composed it.

> Target-action is an Objective-C API, so a non-view-controller controller that uses it must
> inherit from `NSObject`. Unfortunate but necessary — and it is a reason not to push
> target-action down into a view model (see [MVVM](#mvvm)).

---

## Diagnosing an unsustainable controller

**Do not diagnose by line count.** A controller can be under 100 lines, perfectly clean, and still
be the wrong shape. Clean code is a must, but it is not enough — you must also manage dependencies
and responsibilities.

Diagnose by **counting what it coordinates**:

1. Draw the controller's dependency graph — every collaborator, every model it holds, every piece
   of state it manages, every view it configures.
2. Count views, models and pieces of state against the number of controllers. A screen with
   *three views, four models and one controller* has a disparity. That disparity is the signal.
3. Project forward. What will this type also do in six months? Number and date formatting,
   pagination, sharing, animation, analytics events? A type that "seems fine now" but accumulates
   those is already not sustainable.

**Make the count concrete.** Write out every responsibility as its own bullet before and after the
split — the case study's cell controller went from **seven responsibilities to two**, and the
before/after list is what makes the refactor arguable to a reviewer. A line count would have shown
nothing; the controller was already under 100 lines and "pretty clean".

**The fix is more controllers, not less code.** MVC's original goal was many small triads sharing
responsibility. Splitting into tiny MVCs is the fix; extracting one giant helper is not.

Two structural moves make the split stick:

- **Change the table model's type.** A list controller whose model is `[CellController]` instead of
  `[FeedImage]` is decoupled from the domain model *and* from the image loader in one step.
- **Move creation to a composer.** "Objects should not create their own dependencies." A
  controller that only depends on a loader *in order to build another controller* should not
  depend on the loader at all — see `composition-root.md`.

---

## MVVM

MVVM is an MVC variant created at Microsoft to eliminate the boilerplate of syncing view events
and model updates. Its dependency diagram is nearly identical to MVC's, with one difference:
**the view model holds no reference to the view.**

Communication is therefore **indirect, through a binding**. On Microsoft platforms XAML performs
that binding automatically; outside them the pattern is also known as **Model-View-Binder**,
because you write the binder yourself.

### In UIKit, the binder is the view controller

There is no automatic binding in UIKit. That has two consequences worth stating plainly:

- **MVVM saves little boilerplate in UIKit.** You still have a view controller doing the binding —
  there is a tiny MVC inside your MVVM.
- **You do not need Combine or RxSwift for it.** Teams reach for them to get binding; plain
  closures work:

```swift
typealias Observer<T> = (T) -> Void   // naming the type is the documentation
```

### Stateful vs stateless view models

**Stateful** — the view model owns a state machine and notifies on change:

```swift
@MainActor
final class FeedViewModel {
    private enum State { case pending, loading, loaded, failed }
    private var state: State = .pending { didSet { onChange?(self) } }
    private(set) var feed: [FeedImage] = []   // survives a failed reload

    var onChange: Observer<FeedViewModel>?
    var isLoading: Bool { state == .loading }
    var didFail: Bool { state == .failed }

    private let loadFeed: () async throws -> [FeedImage]

    init(loadFeed: @escaping () async throws -> [FeedImage]) {
        self.loadFeed = loadFeed
    }

    func load() async {
        state = .loading
        do { feed = try await loadFeed(); state = .loaded }
        catch { state = .failed }   // error is an extra state; `feed` is untouched
    }
}
```

> **A failed reload keeps what is already on screen.** The error is an *extra* state shown next to
> the content, the loading indicator stops on both success and failure, and the resource is never
> reset to empty on error — "it should not alter the current rendering state on error" (005/004).
> The classic bug is a `.loaded(items)` / `.failed` enum whose `feed` becomes `nil` on failure: a
> binder rendering `vm.feed ?? []` wipes the list on every refresh error. The same applies to a
> SwiftUI `state = .failed`. `LoadResourcePresenter.didFinishLoading(with: Error)` gets this right
> by never calling `resourceView.display`.

**Stateless** — the view model holds only *transient* state: one observer per thing that can
change, and the transition is passed directly through the closure:

```swift
@MainActor
final class FeedViewModel {
    var onLoadingStateChange: Observer<Bool>?
    var onFeedLoad: Observer<[FeedImage]>?

    private let loadFeed: () async throws -> [FeedImage]

    init(loadFeed: @escaping () async throws -> [FeedImage]) {
        self.loadFeed = loadFeed
    }

    func load() async {
        onLoadingStateChange?(true)
        if let feed = try? await loadFeed() { onFeedLoad?(feed) }
        onLoadingStateChange?(false)
    }
}
```

`@MainActor` on the view model keeps every observer callback on the main thread with no `weak
self` dance — the binder (the view controller) is already main-actor-isolated.

For codebases not yet migrated to async/await, the **closure-era original** of the stateless
variant looks like this — same shape, with the loader's completion closure replacing `await`:

```swift
// Closure-era original — keep only while the loader itself is completion-based.
final class FeedViewModel {
    var onLoadingStateChange: Observer<Bool>?
    var onFeedLoad: Observer<[FeedImage]>?

    func loadFeed() {
        onLoadingStateChange?(true)
        loader.load { [weak self] result in
            if let feed = try? result.get() { self?.onFeedLoad?(feed) }
            self?.onLoadingStateChange?(false)
        }
    }
}
```

Stateful comes first naturally, then collapses: once the feed is forwarded through a closure the
`loaded`/`failed` cases are unnecessary, the enum becomes a `Bool`, and the `Bool` becomes
transient. **Prefer stateless** unless the view genuinely needs to re-read state it missed.

### The view model is a value converter

A view model also acts as a **value converter / value transformer**: it formats a domain `Date`
into a user-readable string so the core model stays agnostic of presentation formatting and the
view stays free of transformation logic.

Anything impure that formatting depends on — `currentDate`, `Locale`, `Calendar` — must be
**injectable with sensible defaults**, or the tests are not deterministic. A relative-date test
that passes in `en_US` and fails in `pt_BR`, or fails every leap year, is not a test.

### Keeping the view model platform-agnostic

**The proof is the import.** Delete `import UIKit` from the view model's file. If it still
compiles, it is genuinely reusable on watchOS, macOS and tvOS. If it does not, you are leaking a
UI framework into presentation.

Two rules keep that import out:

- **Generic over the UI type + injected transformer.** Instead of producing a `UIImage`, make the
  view model generic over `Image` and inject `(Data) -> Image?`. The composer passes
  `UIImage.init` on UIKit and a platform equivalent elsewhere. The same trick works for colours
  and any other framework type.
- **Never make a view model an `NSObject` to satisfy target-action.** That is a UIKit requirement
  leaking into presentation. Keep the target-action on the controller and forward the message.

### When is a view model warranted?

> If you find yourself duplicating code across controllers, or performing value transformations in
> controllers, move that logic into a reusable platform-agnostic view model. Otherwise, a simple
> MVC solution will do just fine.

You are not obliged to have a view model just because you have a view — and you can mix MVC and
MVVM in the same UI layer.

---

## MVP

MVP replaces the controller with a **presenter**. The structural difference from MVC:

> In MVC the controller holds a reference to the **concrete** view type. In MVP the presenter
> holds a reference to an **abstract** view type, in the form of a protocol.

**The view protocol belongs to the presentation layer, not the view layer.** If it lived in the
view layer, the presenter would depend on the framework-specific view and the presentation layer
would no longer be cross-platform. This is the single most important placement decision in MVP.

The presenter translates model values into a data structure — variously called a **view model**,
**view data**, or **presentable model** — that holds **only data, no behaviour**. That is the
difference from an MVVM view model, which has dependencies and behaviour.

```swift
// Presentation layer — Foundation only
@MainActor public protocol ResourceLoadingView {
    func display(_ viewModel: ResourceLoadingViewModel)
}

public struct ResourceLoadingViewModel {   // data only — extensible without breaking the protocol
    public let isLoading: Bool
}
```

> Wrap primitives in a view model struct even when there is only one property. `display(isLoading:
> Bool)` cannot grow a `lastUpdated` field without breaking every conformance;
> `display(_ viewModel:)` can.

### Two ways to implement the view in UIKit

1. **Subclass the UIKit control** and give it a presenter reference.
2. **Supervising Controller** — the view controller conforms to the abstract view protocol and
   translates `UIView` events into presenter events. **Prefer this one.**

> "Nothing changes in the model or presentation layers, but the view layer becomes easier to develop
> and maintain as we don't need to subclass custom UIKit views."

### The MVP variation gallery

MVP has more variations than any other pattern here, and they differ only in *how the view reaches
the presenter*. Pick by how much decoupling the situation earns:

| Variation | View → presenter | Use when |
|---|---|---|
| Plain MVP | Direct reference to the concrete presenter | Smallest graph; couples the view to a concrete type |
| Closure | `var onLoad: (() -> Void)?` | One event, no naming needed — "we just need the same type information, void to void" |
| **Presenter-input protocol** | View calls a protocol **owned by the presentation module** | The common form; keeps the presenter agnostic of the UI framework |
| **Presentation adapter (unidirectional)** | View → adapter → domain service → presenter → view | **Preferred here.** The presenter loses all state management and never talks to domain services |
| View-delegate protocol | Protocol owned by the **view** layer; the composition adapter conforms | Also valid — the presenter stays agnostic either way |

The adapter variation is what makes MVP unidirectional and is why this codebase has
`LoadResourcePresentationAdapter`: the presenter shrinks to pure translation, and the view layer
never holds a presenter at all.

> **Read the diagram with the composer deleted.** What remains — view protocol, presenter, view
> model — is the architecture. The rest is wiring.

### The two-way channel and its cost

Unlike MVC, the view talks to the presenter directly. That two-way channel introduces a potential
retain cycle, and it is also why splitting protocols matters: collapsing two observable properties
into one fat view protocol is an **Interface Segregation** violation. Split them
(`ResourceLoadingView`, `ResourceErrorView`, `ResourceView`) — one component can still implement
all three when it renders all three, and you can decompose later when it does not.

### Making MVP unidirectional

Put a **presentation adapter** in the composition layer between the view and the domain service.
The presenter then has no state management and no knowledge of domain services, and the view no
longer talks to the presenter directly:

```
view event → PresentationAdapter → domain service → Presenter.didFinishLoading → view.display
```

Full adapter code: `adapters-and-proxies.md`.

---

## Memory management belongs in the composer

This is the reasoning behind `WeakRefVirtualProxy`, and it generalizes far beyond it.

The naive fix for the MVP retain cycle is `weak var view` on the presenter. Do not do it:

- `weak` requires an `AnyObject` constraint on the view protocol — so a *composition* detail has
  changed the *presenter's* API.
- Even `AnyObject & ResourceLoadingView` still leaks the composition detail.
- It produces false general rules ("should all view protocols be class-constrained in MVP?").
- **Different compositions need different memory rules.** In one composition the view must be held
  weakly; in another (a view adapter the composer owns) it must be held strongly. A rule baked
  into the presenter cannot be right for both.

> Memory management is a responsibility that belongs in the **composer**, not in your components.

So the presenter holds its view strongly and knows nothing about lifetime, and the composer wraps
the view in a proxy when *that particular composition* needs weakness:

```swift
final class WeakRefVirtualProxy<T: AnyObject> {
    private weak var object: T?
    init(_ object: T) { self.object = object }
}

extension WeakRefVirtualProxy: ResourceLoadingView where T: ResourceLoadingView {
    func display(_ viewModel: ResourceLoadingViewModel) { object?.display(viewModel) }
}
```

The same principle licenses the one place this skill permits property injection. MVP composition
is genuinely cyclic — the presenter needs the view, the view needs the adapter, the adapter needs
the presenter — so something must be set after construction. Doing it on the **adapter**, which is
already a composition component, leaks nothing. Doing it on the presenter would.

> **Constructor injection is the default.** Property injection is acceptable *only* in composition
> components, and only to close a cycle that construction order cannot.

If you have no centralized place for composition, you are forced into suboptimal decisions like
property injection *inside your components*. That is the practical argument for a Composition Root.

### Read the diagram without the composer

When an MVP diagram looks complicated, mentally delete the composer and its adapters. What remains
— view protocol, presenter, view model — is the actual architecture. The rest is wiring.

---

## Choosing — and why "none of them" is the honest answer

> Which is best: MVC, MVVM or MVP? **None.** It depends on your current challenges. You should be
> able to refactor from one pattern to the other, and to couple, decouple, compose and decompose
> your UI and presentation layers as needed.

**Start from the clients, not the acronym.** Who renders this presentation logic decides the
pattern:

- **Any client is SwiftUI (including watchOS)** → an `@Observable @MainActor` view model per
  platform, sitting on a Foundation-only presentation mapper (pure functions + strings) that all
  platforms share. SwiftUI binds to observable state; forcing it through view protocols and proxies
  adds wiring it does not need (see `swiftui-composition.md`).
- **Every client is imperative (UIKit/AppKit)** → MVP with view protocols + presentation adapter +
  `WeakRefVirtualProxy` is the default.

Within that, practical guidance:

| Situation | Reach for |
|---|---|
| One platform, no duplication across screens, little formatting | **MVC** — and split into tiny MVCs as it grows |
| Value transformation or logic duplicated across controllers | **MVVM** — extract a platform-agnostic view model |
| Presentation logic reused across imperative clients, or you want the view fully passive and snapshot/unit testable | **MVP** — abstract view protocols in the presentation layer |
| SwiftUI clients alongside (or instead of) UIKit | **Shared mapper + `@Observable` view model per platform** |
| Several features share the same loading/error/success shape | Generic `LoadResourcePresenter` — see below |

Notes that keep this honest:

- **MVP has many variations.** Teams adapt it; you can increase or decrease coupling as needed.
- **Refactoring goes both ways.** When a tiny controller becomes trivial (because a storyboard or
  a composer absorbed its work), merging it back is also a refactor.
- **Do not get too focused on the acronym.** Learn all three and adapt them; there is no one
  solution.

---

## Where presentation strings are created

Customer-facing bundled content is presentation, not UI:

| Pattern | Who creates the string |
|---|---|
| **MVC** | A presentation component, or the Controller/View |
| **MVVM** | The ViewModel |
| **MVP** | The Presenter |

> Localized strings should only appear in an MVC *Model* if that model is itself a view model.

For content that does not vary, a `static` property on the presenter is enough
(`FeedPresenter.title`). **The view must not read it directly** — the composer assigns it, so the
same view controller can be reused with a different title. If the string varies with state, make
it a view-model property the presenter sets per state (and, following ISP, potentially its own
`TitleView`/`TitleViewModel`).

The **UI module is passive**: it makes no presentation decisions, it renders what it is given.
That is what makes it easy to develop, test, reuse and replace.

**Each module owns its strings.** Two production conventions (005/009): a key is never its own
default value (`FEED_VIEW_TITLE`, not `"My Feed"`), and each feature gets its own table in its
presentation module, loaded from that module's bundle. Ownership follows from that (007/002):

- **Several modules depending on the same key is a red flag.** Move shared copy (the generic
  connection error) into the shared presentation module's own table (`Shared.strings`).
- **Expose strings as public statics** (`FeedPresenter.title`, `LoadResourcePresenter.loadError`)
  so composers and integration tests never hard-code keys or table names.
- **White-label apps** reusing the same modules with different copy: move the strings into each
  app's Composition Root and inject them — every app instance has its own root and its own table.

> Asserting keys rather than translations, and the localization test helpers, belong to the
> testing skill.

---

## The generalization arc (feature-specific → generic)

Every generic type in this skill is the **end** of this progression, not the start. Do not begin
here.

1. **Feature-specific MVC.** `FeedViewController` owns the table, the refresh control and the
   cells.
2. **Tiny MVCs.** Extract a refresh controller and a cell controller; the list's model becomes
   `[CellController]`.
3. **Feature-specific presenter (MVP).** `FeedPresenter` + `FeedView`/`FeedLoadingView`/
   `FeedErrorView`, with a composer wiring adapter → presenter → view.
4. **Second feature arrives.** Duplicate it. Only now is the shape of the abstraction visible.
5. **Split combined view models by concern.** A view model mixing static details with loading
   state (`FeedImageViewModel { description, location, isLoading, shouldRetry, image }`) blocks
   reuse. Split it into a details-only view model plus the separate loading, error and resource
   states (`ResourceLoadingView`, `ResourceErrorView`, `ResourceView` displaying the image) — "we
   split the centralized state into different states, which makes our code more composable"
   (007/002). Only then can a per-cell async resource plug into the same generic presenter as the
   list.
6. **Generalize.** The three loading states — start loading, finish with resource, finish with
   error — are identical for every resource, including images. Rename away the feature vocabulary
   (`FeedView` → `ResourceView`, `didStartLoadingFeed` → `didStartLoading`), strip feature-specific
   state (the `title` belongs to each feature, not to the generic presenter), and introduce the one
   thing that varies: a `Mapper`.

```swift
public typealias Mapper = (Resource) throws -> View.ResourceViewModel
```

What varies per feature stays out of the generic types entirely: a shared list or presenter
takes no feature flags (`showsLoadMore`, `isPaginated`) — per-feature behaviour is injected as
content by whoever composes it (see `adapters-and-proxies.md`, load-more as another
`CellController`).

> Make it **concrete before you make it generic**. Do the rename-and-strip refactor with a
> placeholder type first, get green, commit — *then* introduce the generic parameters. Doing both
> at once produces compiler errors that tell you nothing.

The mapper should **throw**, not return an optional: a failed `UIImage(data:)` then surfaces as a
real error state instead of silently rendering nothing.

Two further reductions worth knowing:

- Once everything is a static `map` plus a static `title`, the presenter type **is just a
  namespace** — you can pass a view model's initializer directly as the mapper.
- Adapting `(URL) -> Resource` into `() -> Resource` by capturing the URL at composition time is
  **partial application**. The client never learns the parameter existed; it stays in the
  composition.

Endpoints of this arc, with code: `LoadResourcePresenter` in `architecture-layers.md`,
`LoadResourcePresentationAdapter` and `CellController` in `adapters-and-proxies.md`, composers in
`composition-root.md`.

### SwiftUI

`@Observable` collapses the binder: SwiftUI observes the view model directly, so there is no
adapter → proxy → presenter chain and **no `WeakRefVirtualProxy`** (SwiftUI owns the view
lifecycle). The presentation layer's static mappers, view-model structs and localized strings are
reused unchanged — the boundary rules do not move, only the binding mechanism. Deep coverage —
`@main App` composition root, environment DI, `@Observable` presenters, routing at the root —
lives in [swiftui-composition.md](swiftui-composition.md).

---

## Guardrails

- Do not treat a screen as one MVC — a screen is many small MVCs
- Do not diagnose a bloated controller by line count — count its dependencies, models and state
- Do not put a view protocol in the view layer — it belongs to the presentation layer, or the
  presenter is no longer cross-platform
- Do not import a UI framework in a view model or presenter — inject a transformer closure instead
- Do not make a view model an `NSObject` to satisfy target-action — keep that on the controller
- Do not put `weak var view` on a presenter — wrap the view in a proxy at the composition site
- Do not use property injection outside composition components
- Do not build a generic presenter for one feature — duplicate first, generalize on the second
- Do not let a view read a presenter's static string directly — the composer assigns it
- Do not reset content to empty when a reload fails — show the error as an extra state and stop loading on both paths
- Do not pick a pattern by acronym — pick by the clients (any SwiftUI client → `@Observable` view model over a shared mapper; all-imperative → MVP)
- Do not share one strings table across feature modules — shared copy goes in the shared presentation table

## Verification

- [ ] Deleting `import UIKit`/`import SwiftUI` from every presentation file still compiles
- [ ] Every view protocol lives in the presentation layer and is `@MainActor`
- [ ] View models produced by presenters hold data only — no dependencies, no behaviour
- [ ] No presenter holds a `weak` view reference; proxies are applied in the composer
- [ ] No controller both creates and uses a collaborator it only needs in order to build another
- [ ] Locale/date/calendar dependencies in presenters are injected with defaults
- [ ] Generic presentation types have at least two real clients
- [ ] A failed reload leaves previously rendered content in place (there is a test for it)
