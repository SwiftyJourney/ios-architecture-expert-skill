# Reference Index

Quick navigation for the iOS Architecture Expert skill.

## Architecture & Layers

| File | Use it for |
|---|---|
| `architecture-layers.md` | layer boundaries, domain models, protocols, value types |
| `spm-project-structure.md` | module layout, Package.swift, Swift 6 build settings, CI |

## Composition & Wiring

| File | Use it for |
|---|---|
| `composition-root.md` | FeedService orchestrator, Scheduler, fallback strategy, dependency creation |
| `adapters-and-proxies.md` | LoadResourcePresentationAdapter, WeakRefVirtualProxy, FeedViewAdapter, pagination |

## Concurrency in Architecture

| File | Use it for |
|---|---|
| `concurrency-at-boundaries.md` | Scheduler protocol, `@Sendable` at boundaries, `Task.immediate`, cancellation, `async let` |

## Testing

| File | Use it for |
|---|---|
| `testing-strategy.md` | unit, spec-based, integration, snapshot, acceptance tests |

## Workflow

| File | Use it for |
|---|---|
| `feature-implementation-workflow.md` | step-by-step feature building end-to-end |

## Problem Router

- "I need to structure a new feature module" -> `architecture-layers.md`
- "I need to wire dependencies in the Composition Root" -> `composition-root.md`
- "I need to add pagination or adapt a use case to UI" -> `adapters-and-proxies.md`
- "I need to handle CoreData/concurrency safely" -> `concurrency-at-boundaries.md`
- "I need to test a use case" -> `testing-strategy.md`
- "I need to build a feature end-to-end" -> `feature-implementation-workflow.md`
- "I need to set up SPM modules" -> `spm-project-structure.md`
- "I need to fix a specific architecture issue" -> `../SKILL.md` (diagnostic table)
