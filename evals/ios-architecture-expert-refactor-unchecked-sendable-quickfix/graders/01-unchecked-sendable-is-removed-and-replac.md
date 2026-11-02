---
type: llm
focus: trace
weight: 1
---

@unchecked Sendable is removed and replaced by actor isolation (@MainActor on the view model) -- not retained, and not merely silenced with nonisolated(unsafe), @preconcurrency, or MainActor.assumeIsolated
