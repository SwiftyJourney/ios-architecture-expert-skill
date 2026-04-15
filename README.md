# iOS Architecture Expert Skill

A Claude Code skill that guides building modular, testable iOS/Swift applications following the [Essential Developer](https://www.essentialdeveloper.com) methodology — clean architecture with protocol-oriented design, composition root patterns, generic presenters, and modern Swift concurrency.

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
npx skills add swiftyjourney/ios-architecture-expert-skill
```

## Compatible Agents

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [Cursor](https://cursor.sh)
- [GitHub Copilot](https://github.com/features/copilot)
- [Windsurf](https://codeium.com/windsurf)
- Any agent supporting the `skills/` convention

## What the Skill Covers

### Architecture Layers

- **Feature Layer**: Domain models (`struct`, `Hashable`, `Sendable`) and use-case protocols
- **API Layer**: Endpoint enums, static mappers, `HTTPClient` protocol
- **Cache Layer**: Store protocols, local models, cache policy objects
- **Presentation Layer**: Generic `LoadResourcePresenter<Resource, View>`, framework-agnostic view models
- **UI Layer**: UIKit (`DiffableDataSource`, `CellController`) and SwiftUI (`@Observable`) patterns
- **Composition Layer**: Composition Root, UI composers, adapters, virtual proxies

### Composition Patterns

- `FeedService` — `@MainActor` orchestrator with lazy init, Scheduler, and fallback strategy
- `FeedUIComposer` — Static factory wiring presenter->adapter->view chains
- `LoadResourcePresentationAdapter` — Generic async loader with `Task.immediate` and cancellation
- `WeakRefVirtualProxy` — Retain cycle prevention via conditional conformance
- `Paginated<Item>` — Recursive pagination model with `@Sendable` `loadMore`
- `Scheduler` protocol — Abstract CoreData/InMemory execution context
- `InMemoryFeedStore` — `@MainActor` production fallback + acceptance test double

### Testing Strategy

- Unit tests with `makeSUT()` factory and spy collaborators
- Generic async test spy (`LoaderSpy`) with `AsyncThrowingStream`
- Specification pattern for shared store contract tests
- Integration tests with real CoreData stack
- Snapshot tests (light/dark mode + accessibility sizes)
- Acceptance tests with full DI container

### Concurrency at Architecture Boundaries

- `@MainActor` placement rules (presenters, adapters, composition — not domain types)
- `@Sendable` closures at composition boundaries
- `Task.immediate` for synchronous-first execution
- `async let` for parallel loading in the Composition Root
- Cancellation handling in adapters
- Swift 6 strict concurrency (`SWIFT_STRICT_CONCURRENCY = complete`)

## Skill File Structure

```
SKILL.md                                 # Hub (~180 lines)
references/
├── _index.md                            # Navigation hub
├── architecture-layers.md               # Layer-by-layer patterns with code
├── composition-root.md                  # FeedService, Scheduler, fallback, SceneDelegate
├── adapters-and-proxies.md              # Adapters, proxies, pagination wiring
├── concurrency-at-boundaries.md         # Concurrency patterns at architectural boundaries
├── testing-strategy.md                  # 5-layer test strategy + generic async spy
├── feature-implementation-workflow.md   # Step-by-step feature building
└── spm-project-structure.md             # SPM module layout + Swift 6 settings + CI
```

## Related Skills

- [swiftui-expert](https://github.com/SwiftyJourney/swiftui-expert-skill) — SwiftUI composition, state management, navigation

## Credits

Architecture patterns extracted from the [iOS Lead Essentials](https://www.essentialdeveloper.com/ios-lead-essentials) program by [Essential Developer](https://www.essentialdeveloper.com). Code examples adapted and modernized for Swift 6 / iOS 18+.

## License

MIT — see [LICENSE](LICENSE).
