# Dependency Management — singletons, injection, and composition ownership

Use this when:
- Someone asks "should this be a singleton?" or wants to remove one
- You are deciding between constructor, property and method injection
- A shared API client / service has become a bottleneck every feature depends on
- You are weighing Pure DI against a DI container, or considering a service locator
- Storyboards (or any framework-owned instantiation) are constraining your injection
- You need to read a dependency diagram and say what is wrong with it

Skip this file if:
- You need the wiring code itself. Use `composition-root.md`.
- You need the pattern catalog (adapter/decorator/composite). Use `adapters-and-proxies.md`.
- You need the *diagram notation*. It lives in the `requirements-engineering` skill —
  `references/diagrams.md` documents all four relationship types.

Jump to:
- [Three kinds of "singleton"](#three-kinds-of-singleton)
- [Ambient Context](#ambient-context)
- [Getting out of a singleton](#getting-out-of-a-singleton)
- [The four levels of dependency evolution](#the-four-levels-of-dependency-evolution)
- [The three injection techniques](#the-three-injection-techniques)
- [Constrained Construction](#constrained-construction)
- [Composer rules](#composer-rules)
- [Pure DI, containers, and the Service Locator anti-pattern](#pure-di-containers-and-the-service-locator-anti-pattern)
- [Object lifetimes](#object-lifetimes)
- [Reading a dependency diagram](#reading-a-dependency-diagram)

---

## Three kinds of "singleton"

"Do not use singletons" is too blunt to be useful. Three different things get called a singleton,
and only one of them is the pattern:

| | Shape | Can clients create another? | What it actually is |
|---|---|---|---|
| **Capital-S Singleton** | `private init()` + `static let shared` | **No** — the compiler forbids it | The Gang of Four pattern: "ensure a class has only one instance and provide a global point of access to it" |
| **lowercase-s singleton** | `public init()` + `static let shared` | **Yes** | A conveniently shared instance. `URLSession.shared`, `UserDefaults.standard`. Breaks the GoF contract by design |
| **Mutable global shared state** | `static var shared` | Yes, *and* can replace the global one | **Not a singleton at all.** This is global mutable state |

The third is the dangerous one, and it is usually reached by accident: a team makes `shared` a `var`
so tests can swap it, and the type silently stops being a singleton. Its state can now be changed
from any thread or process in the app, which increases the chance of the system sitting in an
inconsistent state.

### When a Singleton is justified

> When you need **precisely one** instance of a type, and it must be reachable from a well-known
> access point — and having two would be a bug.

A console logger qualifies. **Views, presenters, view models and coordinators do not** — they should
be allocated and deallocated on demand. Singleton objects should be rare and should have a genuine
one-to-one relationship with the system.

> **Testability is not automatically the problem.** There is nothing wrong with having a singleton
> and still having testable code. The problem is *reaching for it directly* from a client — see
> below.

---

## Ambient Context

The pattern behind most mutable-global-state designs: a global `Context` exposing the environment —
current date, networking, database — that any code can read, and that tests can swap.

It is genuinely convenient, and that is its whole appeal: easy access everywhere, easy configuration
of the environment, and an easy way to fake time or network responses in tests. The cost is that
every dependency taken from it is **implicit** — invisible in the initializer, invisible in the
method signature, and impossible for the compiler to check.

Prefer injecting what you need. If you keep an ambient context, treat it as a Composition Root
detail and never let a component read it directly.

---

## Getting out of a singleton

The refactoring is gradual and each step keeps the tests green. Given a view controller calling
`APIClient.shared.login(...)`:

1. **Allow substitution.** Change the client's `let shared` to something the test can replace, or
   add `var api = APIClient.shared` on the client so a test can inject a subclass. *(Transitional —
   you now have global mutable state; do not stop here.)*
2. **Inject the instance.** Constructor-inject the client. The caller no longer *locates* it.
3. **Depend on an abstraction.** Replace the concrete type with a protocol (or a closure) the module
   owns. Now the singleton-ness is invisible to the client and irrelevant to the design.
4. **Move creation to the Composition Root.** It can now be a singleton, a global, `URLSession`, or
   a third-party client — nobody downstream cares.

The core diagnosis, worth stating to anyone who resists the change:

> Reaching for `.shared` mixes **two responsibilities**: invoking a method on an object, and
> *locating* that object. Injecting separates them.

And the practical consequence: a client that reaches for a shared concrete type must know (1) where
to find the instance, (2) its concrete type, and (3) which function to call — *when all it wanted
was a function.*

> **Third-party frameworks ship singletons for convenience.** Hiding one behind an interface you own
> protects you from their breaking changes, makes your code testable, and lets you swap the
> framework later.

---

## The four levels of dependency evolution

How a shared API client typically evolves. Useful both as a diagnosis ("we're at level 2") and as a
roadmap.

| Level | Shape | Problem it still has |
|---|---|---|
| **1** | Every feature calls methods on one shared concrete `APIClient` | Adding `loadFollowers` forces recompiling and redeploying every module. You cannot move one feature to another project without dragging the client with it |
| **2** | A generic client (`execute(request:completion:)`) plus a per-feature `extension APIClient` | Much more flexible, but the client is still shared — a new module can still force a new method and break others |
| **3** | **Invert it.** Each module declares the protocol or closure *it* needs; the concrete client depends on the modules | The client now conforms to many interfaces |
| **4** | Generic client + module-owned interfaces + **adapters** wiring them in the Composition Root | None structurally — but more types and more indirection |

Two things the course is emphatic about:

- **"99% of the applications I've seen stopped at level 2. And eventually when this gets too messy,
  the solution is to rewrite the application."**
- **Do not start at level 4 for a small app.** Move little by little, but be deliberate about the
  direction you are heading.

The reason inversion is the right direction: *the API client is a detail*. The business cares about
features, not about how requests are made. Details should depend on features.

---

## The three injection techniques

| Technique | Use when | Cost |
|---|---|---|
| **Constructor injection** | The default. The client should not know how to find the collaborator, and must have it to function | Requires you to control instantiation |
| **Property injection** | You do *not* control instantiation (storyboards, `NSObject` framework types), or you must close a construction cycle | No compile-time guarantee the dependency was set; the property must be `var` and usually optional |
| **Method injection** | The dependency varies per call, or you want to pass *data* rather than a *function* that produces it | Every call site must supply it |

**Prefer constructor injection**, because it guarantees a component has access to its dependencies at
all times. A presenter's view reference should not be optional — a presenter must have a view.

Method injection has a second, less obvious use: **it removes impurity**. Instead of injecting a
`() -> Date` closure into a policy (making it non-deterministic), pass the `Date` itself:

```swift
// Impure: the policy now depends on when it is called
init(currentDate: @escaping () -> Date)

// Pure: the caller owns the impurity, the policy is deterministic
static func validate(_ timestamp: Date, against date: Date) -> Bool
```

> **Property injection is acceptable only in composition components.** Using it inside a feature
> component leaks a composition detail — see `presentation-patterns.md`.

---

## Constrained Construction

> "Constrained Construction forces all implementations of a certain Abstraction to require their
> constructors to have an identical signature with the goal of enabling late binding… Late binding
> introduces extra complexity, and complexity increases maintenance costs."
> — Seemann & van Deursen, *Dependency Injection: Principles, Practices, and Patterns*

The everyday iOS instance is **storyboard instantiation**: `init?(coder:)` is imposed on you, so you
lose constructor injection and the compile-time composition checks that come with it, and fall back
to property injection.

**But this is no longer forced.** On iOS 13+ you can keep initializer injection while still
instantiating from a storyboard:

```swift
let controller = storyboard.instantiateViewController(identifier: "FeedViewController") { coder in
    FeedViewController(coder: coder, loader: feedLoader)   // real constructor injection
}
```

> The lecture video predates this API and says you *must* move to property injection; the written
> lecture notes correct it. Prefer the `creator:` overload.

Force-casting the result (`as! FeedViewController`) is acceptable in a composer: a failure there is
a developer error, and the composition is covered by tests.

---

## Composer rules

A **Composer** is a type (often a static factory with a private `init`, effectively a namespace)
that creates and wires an object graph. Two rules:

1. **Composers should only be used in the Composition Root.**
2. **Only Composers can use other Composers.**

Factories follow the same containment:

> "[creating instances] isn't suddenly 'illegal,' but you should refrain from using it to get
> instances of Volatile Dependencies outside the Composition Root."
> — Seemann & van Deursen

**A factory does not solve a dependency problem — it moves it.** If a view controller needs a loader
only in order to build a cell controller, introducing a cell-controller factory leaves the view
controller with the same dependency (or more). The fix is to change what the client depends on
— e.g. make the table model `[CellController]` instead of `[FeedImage]` — so the dependency
disappears entirely.

> Abstract Factories can *increase* the number of redundant dependencies and complicate the design.

---

## Pure DI, containers, and the Service Locator anti-pattern

> "Inside the Composition Root you can decide to compose your dependencies manually — that's using
> **Pure DI** — or to delegate it to a **DI Container**… **Accessing a DI Container outside the
> Composition Root results in the Service Locator Anti-Pattern.**"

This skill's codebase uses Pure DI: plain initializers, called in one place. That is a legitimate
end state, not a stepping stone.

If you do adopt a container, the rule is absolute: **only the Composition Root may reference it.**
Otherwise every feature is coupled to the container *and*, transitively, to everything the container
depends on — which is the coupling you introduced DI to avoid.

The same containment applies to **coordinators, flows and routers**. Introduce them when navigation
complexity actually justifies them (they carry their own complexity), and keep them in the
Composition Root or in modules — never inside features.

---

## Object lifetimes

Lifetime is a composition decision, and it is separate from the Singleton *pattern*:

| Lifetime | Example | Enforced by |
|---|---|---|
| **Singleton lifetime** | `HTTPClient`, the CoreData store — created once, shared across features | The Composition Root, by holding one instance |
| **Transient** | `LocalFeedImageDataLoader`, presenters, adapters — created per use | The Composition Root, by creating on demand |

These are *not* singletons: nothing in the compiler enforces one instance, and no component reaches
for a global. They simply happen to be created once.

---

## Reading a dependency diagram

You do not need the full notation to get value from a sketch. Three fast reads:

- **A boundary module has only *incoming* arrows.** Nothing inside it points out. That is what makes
  it substitutable and what lets everything else depend on it safely.
- **The Main / Composition Root module has only *outgoing* arrows.** If any arrow points *at* it,
  a feature depends on the root — a circular dependency waiting to happen.
- **Many arrows converging on one type is a bottleneck for change.** Ask what happens when it
  changes, and remember you cannot see tomorrow's modules on today's diagram.

Coupling is legible from arrow *direction*: arrows pointing the same way at the same level of
abstraction can be decomposed; arrows pointing both ways are coupling; every module pointing at
every other is the spaghetti case.

> **Diagrams are a framework to think, not documentation to maintain.** Nothing regenerates them
> from source, so they rot. Use them for a 5–10 minute whiteboard session at the start of a feature,
> and annotate only as much as adds value — a diagram nobody can read has no value.

For the exact four-relationship notation (solid vs dashed lines, open vs filled arrowheads, and
what each means in code), see the `requirements-engineering` skill's `references/diagrams.md`.

---

## Guardrails

- Do not reach for `.shared` from a component — inject the dependency and let the root decide what it is
- Do not turn a `static let shared` into a `static var` to make code testable — that is global mutable state, not a singleton
- Do not make Views, presenters, view models or coordinators singletons
- Do not stop at level 2 and assume you are done — know which level you are on and why
- Do not use property injection outside composition components
- Do not reference a DI container, coordinator or router from a feature module
- Do not introduce a factory to solve a dependency problem — change what the client depends on
- Do not accept "storyboards force property injection" — use `instantiateViewController(identifier:creator:)`

## Verification

- [ ] No component reads a global or `.shared` instance directly
- [ ] Every collaborator arrives through an initializer, except where a composition cycle forces property injection on a composition component
- [ ] Exactly one place in the app creates concrete infrastructure
- [ ] No feature module imports the Composition Root, a DI container, or a router
- [ ] Impure dependencies (date, locale, calendar) are injected, with defaults where that does not break modularity
- [ ] The dependency diagram has a boundary module with no outgoing arrows and a root with no incoming ones
