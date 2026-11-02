# Design Principles — the reasoning behind the patterns

Use this when:
- You are reviewing unfamiliar code and need to name what is wrong with it
- You must decide *whether* a boundary, protocol, policy type or module is justified
- A type is accumulating responsibilities and you need the argument for splitting it
- Someone asks "why is it built this way?" about any pattern in this skill

Skip this file if:
- You need code for a specific layer. Use `architecture-layers.md`.
- You need pattern implementations. Use `adapters-and-proxies.md`.
- You need MVC/MVVM/MVP guidance. Use `presentation-patterns.md`.

Jump to:
- [Symptom → principle](#symptom--principle)
- [Three kinds of logic](#three-kinds-of-logic)
- [Dependency Inversion anatomy](#dependency-inversion-anatomy)
- [Dependency Rejection](#dependency-rejection)
- [Entities vs Value Objects](#entities-vs-value-objects)
- [Command–Query Separation](#commandquery-separation-as-a-bloat-diagnostic)
- [Interface Segregation](#interface-segregation)
- [Composite Reuse & Liskov](#composite-reuse--liskov)
- [Coincidental duplication vs DRY](#coincidental-duplication-vs-dry)
- [Data Transfer Model Representations](#data-transfer-model-representations)
- [Bottleneck-for-change analysis](#bottleneck-for-change-analysis)
- [Duplicate before you abstract](#duplicate-before-you-abstract)
- [Deterministic core, side effects at the boundary](#deterministic-core-side-effects-at-the-boundary)
- [Make invalid states unrepresentable](#make-invalid-states-unrepresentable)
- [Functional Core, Imperative Shell](#functional-core-imperative-shell)
- [Low coupling, high cohesion](#low-coupling-high-cohesion--the-litmus-test)
- [Access control as a decoupling tool](#access-control-as-a-decoupling-tool)
- [What modularity actually buys](#what-modularity-actually-buys)

---

## Symptom → principle

| What you see | What it means | Section |
|---|---|---|
| A use case both returns data and mutates state | Query with side effects | [CQS](#commandquery-separation-as-a-bloat-diagnostic) |
| A protocol conformance with empty method bodies | The client is forced to implement what it does not need | [ISP](#interface-segregation) |
| A single-method protocol with no named-contract need | A function type would compose with plain code | [Dependency Rejection](#dependency-rejection) |
| A domain model with `Codable`/`@Model`/`CodingKeys` | Framework detail in a framework-agnostic type | [DTOs](#data-transfer-model-representations) |
| Many types pointing at one type in the graph | Bottleneck for change | [Bottleneck analysis](#bottleneck-for-change-analysis) |
| Business rule inlined in a loader/controller | App-agnostic logic living in app-specific code | [Three kinds of logic](#three-kinds-of-logic) |
| `(Data?, Response?, Error?)` or paired optionals | Invalid states are representable | [Invalid states](#make-invalid-states-unrepresentable) |
| A generic type with one client | Abstraction invented before the second example | [Duplicate first](#duplicate-before-you-abstract) |
| A test that knows a business constant (`7 days`) | No single source of truth | [Deterministic core](#deterministic-core-side-effects-at-the-boundary) |
| Two identical tests in different test classes | Probably *not* duplication | [Coincidental duplication](#coincidental-duplication-vs-dry) |
| `UUID`/`String`/`URL` threaded through many signatures | Primitive obsession | [Invalid states](#make-invalid-states-unrepresentable) |

---

## Three kinds of logic

Every line of code in an app is one of three things. Mixing them is the root cause of most of the
symptoms above.

| Kind | What it is | Where it lives | Deterministic? |
|---|---|---|---|
| **Application-specific** | Use cases — the *recipe*. "Delete the old cache, then insert the new one with a timestamp." | Use-case types (also called interactors, controllers, control boundaries, model controllers) | No — orchestrates side effects |
| **Application-agnostic** | Business models and policies — rules that hold regardless of *this* app. "A cache older than the max age is expired." | Domain models, policy types | Yes |
| **Framework** | Infrastructure — how bytes actually move. CoreData, URLSession, FileManager. | Infrastructure implementations | No — is the side effect |

Two consequences:

- **Frameworks obey, they don't decide.** A store implementation never contains "only insert if
  the timestamp is less than seven days old". It inserts what it is told to insert. If a store
  starts making decisions, business logic has leaked into infrastructure — and infrastructure
  implementations become trivial (a handful of lines each) once it hasn't.
- **You dictate the architecture; you plug frameworks in.** Not the reverse.

**Application details** — hiding a store behind an interface, deciding how to get the current
date, dealing with asynchrony — are irrelevant to the core domain model. They are details, and
they belong to the use-case layer, not the model.

> This is why `FeedCachePolicy` is a separate type from `LocalFeedLoader`: the *expiry rule* is
> app-agnostic and deterministic; the *orchestration* around it is not. See
> `architecture-layers.md`.

---

## Dependency Inversion anatomy

Three parts, always in this order:

```
high-level module  ──►  boundary (protocol or closure)  ◄──  low-level implementation
```

Without the boundary, the high-level module talks to the concrete type directly — and inherits
every dependency that type has. **Transitive contagion is the real cost:** if the concrete client
depends on a third-party networking library, so does your use case, and so does anything that
depends on your use case.

The reliable test is **the direction of the arrows in the dependency graph**:

> There are no arrows pointing *to* infrastructure. Infrastructure depends on the business logic.
> Infrastructure components are plugins.

If you can drag the infrastructure box out of the diagram and no arrow snaps, the inversion is
real. If an arrow snaps, it is decoration.

A boundary is well-shaped when it **trades in its own currency**. `HTTPClient` deals in `Data` and
`HTTPURLResponse` — never in a domain model. That is what lets it move into its own module, be
shared across features, and never break when the domain changes.

A shared feature boundary also **does not know where its data comes from.** Anything specific to
one implementation stays with that implementation:

- **URLs and store paths are constructor-injected** into the concrete type, never method parameters
  on the abstraction. The test: does putting it in the signature force every client to know a
  detail of one strategy? (`load(from url:)` on a feed boundary does.)
- **Source-specific error types** (`connectivity`, `invalidData`) live with the implementation; the
  shared boundary throws plain `Error`. The case study's async code lets `HTTPClient` errors
  propagate and maps *any* error to one presentation message, so the rule is "no
  provenance-specific types in shared contracts", not "always wrap errors".

**Ownership follows the clients.** A protocol lives with the code that calls it, not with its
implementations — and when the clients move, the protocol moves. Once no presenter referenced
`FeedImageDataLoader`, it moved from presentation to the feature module (005_007).

**Messy boundaries, clean core.** Infrastructure is allowed to be ugly — threads, disks, sockets.
That does not license an ugly codebase. The core stays side-effect-free; the mess is contained at
the edges.

---

## Dependency Rejection

The step past dependency injection: **do not inject the dependency — compose it away.**

> A protocol with one method can always be replaced with a closure… sometimes you want to name
> things to enforce some kind of contract. (001_2)

The trade-off is **named contract vs anonymous signature**, not the number of strategies. A
single-method boundary defaults to a function type, even with several implementations, because
closures compose fallback, decoration and interception with plain code. A protocol earns its place
when the boundary has **several cohesive methods** (`FeedStore`: delete/insert/retrieve), when a
**named contract** matters to implementers (shared contract specs, external clients), or when
**type-level conformance** carries meaning (`ResourceView` and its `associatedtype`). `FeedLoader`
had remote, local and fallback implementations and was still deleted (commit `fa5568b`): once the
pieces composed with universal abstractions, the named single-method protocol only added a name, a
file, a conformance and a fake in every test.

Instead, compose the pieces at the Composition Root into a plain function:

```swift
// Not a protocol — a composed closure. The closure IS the abstraction.
func loadFeed() async throws -> [FeedImage] {
    let (data, response) = try await httpClient.get(from: url)
    return try FeedItemsMapper.map(data, from: response)
}
```

The consumer receives `() async throws -> [FeedImage]`. It cannot know or care whether that came
from the network, a cache, or a composite of both.

### The functional sandwich

The shape this produces, over and over:

> **side effect → pure function → side effect**

Make the HTTP request (side effect), map the bytes (pure), store to the cache and update the UI
(side effect). All the interesting logic sits in the pure middle, where it is trivially testable
with no doubles at all.

Composition operators replace injection: `map`, `flatMap`, `catch`/`do-catch`. You do not inject
dependencies, you **compose** them.

### The cost side

Rejection is not free — you trade compile-time guarantees for composition flexibility. Two closures
of the same type can be swapped by mistake in a way two protocols could not. **What do you do when
you don't have compile-time guarantees? You write tests.** Make that trade knowingly.

> Applied in this skill: `architecture-layers.md` forbids single-method loader protocols;
> `composition-root.md` shows the composed closures that replace them.

---

## Entities vs Value Objects

| | Entity | Value Object |
|---|---|---|
| Has identity | Yes — a customer is *this* customer | No — a policy is a policy |
| Two with equal fields | Different things | The same thing |
| Implies | An instance, possibly mutable state | A `struct`, or a static namespace |

```swift
// Value Object — compared by its values. Two of these with equal fields ARE the same money.
struct Money {
    let amount: Decimal
    let currency: Currency
}

// Entity — compared by identity. Two of these with the SAME id are identical
// even when their values differ (a balance changes; the account is still the account).
struct Account {
    let id: AccountID
    var balance: Money
}
```

Both can hold business rules; the distinction is only whether identity exists. A Value Object is
often called a "Value Type" or "just data".

A **policy has no identity and no state**, so it should not be instantiable:

```swift
final class FeedCachePolicy {
    private init() {}                                    // never instantiate a rule
    private static let calendar = Calendar(identifier: .gregorian)
    private static var maxCacheAgeInDays: Int { 7 }

    static func validate(_ timestamp: Date, against date: Date) -> Bool { ... }
}
```

The age limit is `private` because it is a business decision that will change (7 days today,
14 next quarter). Callers ask the policy a yes/no question and never see the number. Tests express
boundaries *relative* to the policy through one helper (the case study's `minusFeedCacheMaxAge()`
test extension holds the only copy of the 7), so changing the rule edits the policy and that helper,
not every suite that mentions "7 days".

Choose the namespace form that fits: a `final class` with a private `init`, a `struct`, or an
`enum` with no cases (which cannot be instantiated at all, so it needs no private init). In its own
module it could simply be three free functions.

Rules expressed in days use `Calendar.date(byAdding: .day, …)` with an explicit (Gregorian)
calendar, never `n * 24 * 60 * 60` — not every day has 24 hours across DST changes. If the calendar
returns `nil`, the policy answers "invalid" (`false`) rather than crashing (003_006).

---

## Command–Query Separation as a bloat diagnostic

CQS says: separate querying from commands with side effects. Its real value here is as a
**detector for a use case doing too much**.

The tell:

1. A method named like a query (`load`) also mutates state (deletes an expired cache).
2. The requirement that justifies the mutation is not written anywhere — the word "invalidate"
   appears nowhere in the spec. It is **implicit and hidden**.
3. That hidden notion is where the complexity and the side effects came from.

The fix is a **requirements change first, code second**. Recommend confirming with product or the
team whether `load` is supposed to delete at all — a reviewer cannot settle a hidden requirement by
moving code. Then split into two use cases —
`load` (a pure query) and `validateCache` (a command). Then:

- Each is simple enough to test in isolation.
- You can **schedule them independently** — validate on entering background, load on refresh.
- Neither has to explain the other.

Flag every behaviour change the split introduces (e.g. `load` now throwing on a retrieval error
instead of returning `[]`) as something the team approves, not as the reviewer's preference.

> The case study shipped deletion *inside* `load`, then reverted it when Load and Validate became
> separate use cases. Code in `architecture-layers.md`.

Corollary: **queries are much easier to test than commands.** Side effects overlap combinatorially
— delete affects insert, insert affects delete, retrieve is affected by both — and that gets worse
under concurrency. Every query you keep pure is complexity you never have to test.

---

## Interface Segregation

**The violation has a visible signature: empty method implementations.** If a conformance has a
method whose body is empty, or whose only purpose is to satisfy the compiler, the client is being
forced to implement something it does not need.

Heuristics:

- **Keep each boundary to one cohesive contract.** Every time a protocol grows a method, ask whether
  it knows too much. Multi-method boundaries are legitimate when the methods genuinely belong to one
  contract — `FeedStore`'s delete/insert/retrieve are one storage contract. Two unrelated operations
  are two boundaries, and a boundary that shrinks to a single method usually becomes a function type
  (see Dependency Rejection).
- **Share a boundary when clients need the same operations; split when the operations differ.**
  One `HTTPClient` serves every remote loader because they all just GET. `FeedStore`
  (delete/insert/retrieve with a timestamp) and `FeedImageDataStore` (insert/retrieve by URL) stay
  separate even though `CoreDataFeedStore` implements both. Add an operation only when a use case
  asks for it (006_001).
- **A method that forces implementations to hold state is a design smell.** A `cancel(url:)`
  method obliges every implementation to remember previous requests. Return a cancellable token
  instead and move state management to the client.
- **Segregate optional behaviour into separate protocols.** Not every store can deterministically
  fail, so `FailableRetrieve…`/`FailableInsert…`/`FailableDelete…` are separate from the base
  contract, and only implementations that can simulate the failure conform.
- **Default implementations in a protocol extension** make a requirement effectively optional —
  the honest way to express "mandatory: create the view; optional: preload/cancel".
- **Struct composition beats protocol composition** when a client needs several unrelated
  capabilities. A `typealias A & B & C` forces one type to implement all three; a struct holding
  three optionals lets each capability be supplied independently. That is the reason `CellController`
  is a struct — see `adapters-and-proxies.md`.

ISP is what makes code **composable and decomposable**: two segregated protocols can still be
implemented by one type when one component renders both, and split apart the day it does not.

---

## Composite Reuse & Liskov

**Prefer protocol inheritance and extensions over class inheritance.** Class inheritance is not
composable — a type inherits from exactly one class — so it cannot express "base contract +
failable-insert + failable-delete but not failable-retrieve". Protocol inheritance plus
composition can:

```swift
protocol FailableRetrieveFeedStoreSpecs: FeedStoreSpecs { ... }
typealias FailableFeedStoreSpecs =
    FailableRetrieveFeedStoreSpecs & FailableInsertFeedStoreSpecs & FailableDeleteFeedStoreSpecs
```

**Liskov Substitution is what makes shared contract specs possible at all.** Because every
implementation is a subtype of the boundary, one set of assertions written against the *protocol*
validates every implementation. A helper that takes `sut: FeedStore` rather than a concrete type is
the practical expression of this — and swapping one store implementation for another with all
tests still green is the proof.

---

## Coincidental duplication vs DRY

Two pieces of code that look identical are only duplication if they will **change for the same
reason**. The keyword is **context**.

- The same "does not message the store on creation" test in `CacheFeedUseCaseTests` and
  `LoadFeedFromCacheUseCaseTests` is **not** duplication — two distinct contexts that will evolve
  independently. Keep both.
- A `FeedStoreSpy` copy-pasted into two test files **is** duplication — one context, one reason to
  change. Extract it.
- `LocalFeedImage` mirroring `FeedImage` is **not** duplication — see below.

Deduplicating coincidental duplication is how you accidentally couple two things that had no
business being coupled.

---

## Data Transfer Model Representations

Each module gets its own representation of the same concept: `FeedImage` (domain),
`LocalFeedImage` (cache), a private `RemoteFeedItem` (API), a private `ManagedFeedImage` (CoreData).

> It might look like we are duplicating code, but they change for different reasons. Separating
> them lets each change at its own pace.

**Each module names its types in its own language.** Domain types use the domain experts'
vocabulary (`FeedImage`, `url`); the API module's private DTOs keep the backend's (`RemoteFeedItem`,
`image`). A naming mismatch between teams is something to resolve deliberately — never paper over
it in a shared model (003_005).

The forces that make this pay:

- **A type can only conform to a protocol once.** A shared model cannot have two different
  `Decodable` implementations for two different APIs. Whoever gets there first wins, forever.
- **`CodingKeys` are an API detail.** Mapping `image` → `imageURL` in a shared model means an API
  rename can break the UI and database modules.
- **Persistence conformances are framework details.** Adding `Codable` (or `@Model`) to a
  cache-layer model tells the next developer it is acceptable to add CoreData or Realm details
  there too. *It is almost not about right or wrong — it is about the message you leave behind.*

The mapping cost is real but small (private `toLocal()`/`toModels()` array extensions). **Measure
before optimizing it**; if you have no measured problem, optimize for maintainability.

> "It's just one string" is exactly where discipline is tested. Small leaks have architectural
> consequences.

---

## Low coupling, high cohesion — the litmus test

Aim for **low coupling between modules and high cohesion within one**. The test is a single question:

> **"Can I refactor or add new features to this module without touching any files of the other
> module?"** Aim for a yes.

And the failure signal is behavioural rather than structural:

> **The design fights change.** When developers "think twice, three, four times before making such a
> costly change", the design is not facilitating change. That hesitation is the measurement — act on
> it early, before the bottleneck sets.

## Bottleneck-for-change analysis

Module-level dependency diagrams lie by omission — they are "surveying the landscape from an
aeroplane". Draw the **component-level** graph: every type, every arrow.

Then look for **many arrows converging on one type**. Ask: what happens if this changes? A domain
model with five incoming dependencies means adding one property risks breaking five modules — and
you do not know what module will exist tomorrow.

Protocols multiply this: a protocol that references a domain model drags *every future
implementation* into that dependency.

The remedies, in order of preference:
1. Give the boundary its own currency types (the `HTTPClient` approach).
2. Give each module its own representation (DTOs).
3. Move the shared thing to the Composition Root, which nothing depends on.

Without a diagram, these bottlenecks are effectively invisible — and invisible bottlenecks are what
make teams stop changing code.

---

## Duplicate before you abstract

> If you don't duplicate the code, you're probably going to come up with the wrong abstraction.

The rhythm:

1. **Copy** the working component for the second feature. Get its tests green.
2. **Make the two copies identical** — push feature-specific bits (a title, a status-code rule,
   stray mapping) out until the only remaining difference is the thing that should be injected.
3. Introduce the variation point with a **concrete placeholder** (a closure, a `String`).
4. **Then** make it generic. One step at a time — doing the rename and the generalization together
   produces compiler errors that tell you nothing.
5. Swap each feature over, one commit each. Delete the now-dead concrete logic.

Copy-paste-then-change is defended here as a *technique*, not a lapse: you have the compiler, the
tests, and git to revert to green. Two working implementations side by side make the right
abstraction obvious; one implementation plus imagination does not.

**Never create a generic type for a single client.** If you *know* the app will have many resources
you may start generic — but that is a prediction, and predictions are where speculative complexity
comes from.

---

## Functional Core, Imperative Shell

The name for the shape the previous sections keep producing:

```
        ┌──────── Imperative Shell ────────┐
        │  stateful components, side        │
        │  effects, frameworks, UI, I/O     │
        │   ┌──── Functional Core ────┐     │
        │   │  side-effect-free       │     │
        │   │  business logic         │     │
        │   └─────────────────────────┘     │
        └───────────────────────────────────┘
```

The payoff is stated as a property, not a preference: **because there are no side effects in the
Functional Core, it is deterministic** — the same input always produces the same output. That is what
makes it trivial to build, maintain and test, and it is why `FeedCachePolicy` takes a `Date` rather
than a `() -> Date`.

The shell is allowed to be messy — threads, disks, sockets. Containing the mess is the job; removing
it is impossible.

## Deterministic core, side effects at the boundary

An **impure** function returns a different value each call. `currentDate()` is impure and
non-deterministic; a model that calls it cannot be reasoned about or tested without controlling
time.

The move is to **push the impurity outward**, from the model into the type that orchestrates:

```swift
// Impure dependency injected into the model — the model is now non-deterministic
init(maxAge: Int, currentDate: @escaping () -> Date)

// Method injection — the caller supplies the value; the model is pure again
static func validate(_ timestamp: Date, against date: Date) -> Bool
```

Initializer injection → **method injection** is the standard cure: instead of passing a *function*
that produces the data, pass the *data*. Side effects then happen only at the boundaries, in the
controller types that deal with impure functions, and the models stay deterministic and reusable.

This has a concurrency dividend: **side effects are the enemy of concurrency.** The fewer you have,
the more of your system can run in parallel safely.

### Single source of truth

A business constant must exist in exactly one place — and the tests must not be the second place.

- Keep the constant **private** to its policy. External modules depend on the abstract question
  ("is it expired?"), never on "seven days".
- Give tests a **domain-named DSL** (`Date.minusFeedCacheMaxAge()`) so they assert semantics.
- Name the concept **abstractly** — "max age", not "max days" — so switching to hours or a
  server-driven value changes nothing else.

Done right, changing the policy from 7 days to 3 requires no change to any test name or test body.

---

## Make invalid states unrepresentable

> If something should never happen, it should not be representable in your code.

**Optionals multiply.** Two optionals give four representable states; three give eight. Most are
invalid, and every one of them is a branch someone has to handle — or forget to handle:

```swift
// 4 representable states, 2 of them nonsense
completion(data: Data?, response: HTTPURLResponse?, error: Error?)

// 2 states, both valid, exhaustively switchable
enum Result { case success(Data, HTTPURLResponse); case failure(Error) }
```

Prefer standard types — `Result`, `Optional` — over bespoke enums: they compose with everything
else that speaks the same language, and `map`/`flatMap` remove branching entirely. An enum whose
cases mean exactly "absence" and "presence" *is* `Optional`; say so. A completion taking `Error?`
is an anti-pattern — it conveys nothing and composes with nothing; use `Result<Void, Error>` or
`throws`.

**Collapse multiple completion call sites.** Several branches each invoking the callback is a
bug source (the classic missing `else` that fires the completion twice). One exit point cannot
have that bug.

**Exhaustive `switch` vs `default` is a trade-off.** Listing every case of a stable enum makes the
compiler flag every site when a case is added; use `default` only when the enum churns and every
new case should land in the same branch (003_008).

**Primitive obsession** is the same failure at a different scale: `UUID`, `String` and `URL`
threaded through many signatures carry no meaning and can be swapped by mistake. A domain type or
an endpoint enum that takes the model restores the compiler's help.

---

## Access control as a decoupling tool

Access control is not bookkeeping — it is where you decide the module's contract.

- `public` is a **deliberate declaration**: this is what other modules may depend on. Everything
  else is `private` or left `internal` (the default — do not write it).
- `final` by default unless you have a concrete use case for subclassing.
- **Keeping a constant private is a design decision**, not tidiness — it prevents other modules
  from depending on an internal detail (see single source of truth above).
- **A type that becomes `public` should be tested in isolation.** While a type is internal it can
  be covered through integration with its client; once it is part of the module's contract, a
  failure in it must point at it, not at whatever composed it.

---

## What modularity actually buys

Modularity is not an aesthetic. The operational definition:

> **Freedom to change an implementation on demand without affecting the rest of the system.**

Concretely: parallel development across teams; a change in the database module that does not force
recompiling and redeploying every other module; the ability to move a module into another project
without dragging a dependency chain with it; and the ability to delete a whole vertical slice.

The counterweight, stated as plainly:

- **Composition gets more complex** as you decompose — which is why the whole composition must be
  tested too.
- **Physical separation costs maintenance.** Start with folders; split into targets, projects or
  repositories only when you have the problem that solves. See `spm-project-structure.md`.
- Monolith versus modular is **an engineering decision you make**, per situation. For a simple use
  case, the concrete monolithic approach may be correct.

> Every one of these options has merits and disadvantages. Nothing is perfect. The skill is
> managing the trade-offs, not memorising a winner.

---

## Guardrails

- Do not add a single-method protocol where a function type would do — keep protocols for cohesive multi-method contracts, named contracts, or meaningful conformance
- Do not put business rules in infrastructure — frameworks obey, they don't decide
- Do not let a query mutate state — split the command out and fix the requirement first
- Do not deduplicate code that shares a shape but not a reason to change
- Do not add framework conformances (`Codable`, `@Model`, `Decodable`) to framework-agnostic models
- Do not build a generic type for one client
- Do not represent states that cannot validly occur
- Do not let tests know business constants — hide them behind a domain-named DSL
- Do not treat any design decision as final; requirements and knowledge both keep changing

## Verification

- [ ] Every protocol has several cohesive methods, a named-contract reason, or meaningful conformance; single-method boundaries are function types
- [ ] Shared boundaries carry no provenance details (URLs, paths, source-specific error types)
- [ ] Infrastructure types contain no conditional business logic
- [ ] Business rules live in stateless, deterministic policy types
- [ ] No conformance in the codebase has an empty method body
- [ ] Each module has its own representation of shared concepts; no shared model has a framework conformance
- [ ] The component-level dependency graph has no type with a suspicious fan-in
- [ ] No generic type has fewer than two real clients
- [ ] Business constants appear exactly once in production and zero times in tests
