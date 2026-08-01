# iOS Architecture Expert Agent Skill

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Agent Skills](https://img.shields.io/badge/Agent-Skills-blue)](https://agentskills.io)

A Claude Code skill that guides building modular, testable iOS/Swift applications following a clean architecture methodology — protocol-oriented design, composition root patterns, generic presenters, and modern Swift concurrency.

## Who is this for?

iOS developers who want AI-assisted guidance on:

- Structuring apps with clean architecture layers
- Composing dependencies via a Composition Root
- Writing testable code with protocol boundaries
- Building reusable presentation logic with generic presenters
- Applying modern Swift concurrency (async/await, `Sendable`, `@MainActor`)
- Enabling Swift 6 strict concurrency across all modules

## Installation

Install via [skills.sh](https://skills.sh):

```bash
npx skills add SwiftyJourney/ios-architecture-expert-skill
```

## Compatible Agents

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [Cursor](https://cursor.sh)
- [GitHub Copilot](https://github.com/features/copilot)
- [Windsurf](https://codeium.com/windsurf)
- Any agent supporting the `skills/` convention

## What the Skill Covers

### Design Principles

The reasoning an agent needs to diagnose unfamiliar code, not just replicate a shape:

- Three kinds of logic — application-specific, application-agnostic, framework
- Dependency Inversion anatomy, and Dependency Rejection (compose instead of inject)
- Command–Query Separation as a detector for a bloated use case
- Interface Segregation, Composite Reuse, Liskov as the enabler of shared contract specs
- Coincidental duplication vs DRY, Data Transfer Model Representations
- Bottleneck-for-change analysis; make invalid states unrepresentable

### Architecture Layers

- **Feature Layer**: Domain models (`struct`, `Hashable`, `Sendable`) and use-case protocols
- **API Layer**: Endpoint enums, static mappers, `HTTPClient` protocol
- **Cache Layer**: Store protocols, local models, cache policy objects
- **Presentation Layer**: Generic `LoadResourcePresenter<Resource, View>`, framework-agnostic view models
- **UI Layer**: UIKit (`DiffableDataSource`, `CellController`) and SwiftUI (`@Observable`) patterns
- **Composition Layer**: Composition Root, UI composers, adapters, virtual proxies

Plus module organization: onion architecture, horizontal vs vertical slicing, and how physically
separated your modules should actually be.

### Presentation Patterns

- **MVC** as *many tiny MVCs* per screen — and diagnosing a controller by dependency count, not lines
- **MVVM** — Model-View-Binder, stateful vs stateless view models, value conversion, cross-platform view models
- **MVP** — abstract view protocols owned by the presentation layer, Supervising Controller, presentation adapters
- Why memory-management policy belongs to the composer (and therefore why `WeakRefVirtualProxy` exists)
- Choosing between them — and the honest answer that it depends on the problem you have

### Dependency Management

- The **three kinds of "singleton"** — the GoF pattern, a shared instance, and mutable global state
- The **four levels of dependency evolution**, and why most codebases stall at level 2
- Constructor / property / method injection, and Constrained Construction
- Composer rules, Pure DI vs DI containers, and the Service Locator anti-pattern

### Infrastructure

- **Never pre-flight a connectivity check** — configure the session and handle the result
- `URLSession` delegate retention, and the representable-states technique
- CoreData details: binary storage, bundles, `/dev/null` test stores
- Logging, `assertionFailure` vs `print`, and graceful degradation with a Null Object

### Composition Patterns

- **Adapter / Decorator / Composite / Interception / Null Object** — when each applies, and when a language construct already does the job
- `FeedService` — `@MainActor` orchestrator with lazy init, Scheduler, and fallback strategy
- `FeedUIComposer` — Static factory wiring presenter->adapter->view chains
- `LoadResourcePresentationAdapter` — Generic async loader with `Task.immediate` and cancellation
- `WeakRefVirtualProxy` — Retain cycle prevention via conditional conformance
- `Paginated<Item>` — Recursive pagination model with `@Sendable` `loadMore`
- `Scheduler` protocol — Abstract CoreData/InMemory execution context
- `InMemoryFeedStore` — `@MainActor`, NSCache-backed production fallback when CoreData fails to init

### Testing Strategy

- How much to test at which level — and why this codebase has no XCUITest target
- Testing through the public API, and the one place `@testable` is legitimate
- Unit tests with `makeSUT()` factory and spy collaborators
- Generic async test spy (`LoaderSpy`) with `AsyncThrowingStream`
- Specification pattern for shared store contract tests
- Integration tests with real CoreData stack
- Snapshot tests (light/dark mode + accessibility sizes)
- Acceptance tests with full DI container
- The Swift Testing delta — including a verified `trackForMemoryLeaks` built on a test-scoping trait

### Concurrency at Architecture Boundaries

- `@MainActor` placement rules (presenters, adapters, composition — not domain types)
- `@Sendable` closures at composition boundaries
- `Task.immediate` for synchronous-first execution
- `async let` for parallel loading in the Composition Root
- Cancellation handling in adapters
- Swift 6 strict concurrency (`SWIFT_STRICT_CONCURRENCY = complete`)

## Skill File Structure

```
skills/ios-architecture-expert/          # the skill — this is what ships
├── SKILL.md                             # Hub + diagnostic table + gotchas + reference router
└── references/
    ├── design-principles.md             # Symptom→principle table; the reasoning behind the patterns
    ├── dependency-management.md         # Singleton taxonomy, DI techniques, composer rules, lifetimes
    ├── infrastructure-and-networking.md # Connectivity, URLSession config, CoreData details, logging
    ├── architecture-layers.md           # Layer-by-layer patterns with code
    ├── presentation-patterns.md         # MVC / MVVM / MVP, and choosing between them
    ├── composition-root.md              # What the root is, lifetimes, FeedService, Scheduler, fallback
    ├── adapters-and-proxies.md          # Adapter, proxy, composite, decorator, interception, pagination
    ├── concurrency-at-boundaries.md     # Why boundaries stay sync + async migration playbook
    ├── testing-strategy.md              # 5-layer test strategy + generic async spy + Swift Testing delta
    ├── feature-implementation-workflow.md  # Feature building, spy-first/protocol-last, legacy extraction
    └── spm-project-structure.md         # Onion, slicing, SPM module layout + Swift 6 settings + CI
evals/                                   # Trigger eval queries + output-quality test cases (repo-only)
.claude-plugin/                          # plugin + marketplace manifests (repo-only)
```

## Related Skills

- [Requirements Engineering](https://github.com/SwiftyJourney/requirements-engineering-skill) — Turn vague requirements into testable specs: BDD, use cases, contracts, diagrams
- [SwiftUI Expert](https://github.com/SwiftyJourney/swiftui-expert-skill) — SwiftUI view code: state, composition, performance, animations, accessibility
- [SwiftUI ⇄ UIKit Interop](https://github.com/SwiftyJourney/swiftui-uikit-interop-skill) — Bridge SwiftUI and UIKit: hosting controllers, representables, data across the boundary
- [Swift Language Expert](https://github.com/SwiftyJourney/swift-language-expert-skill) — Non-obvious Swift language & standard-library techniques

## Credits

Built on well-established clean architecture, TDD, and modular design practices from the iOS community. Code examples written for Swift 6 / iOS 18+.

## License

MIT — see [LICENSE](LICENSE).
