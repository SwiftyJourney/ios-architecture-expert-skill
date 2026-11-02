---
type: llm
focus: trace
weight: 1
---

The network completion closure is handled so that state is mutated on the main actor even if APIClient calls back on a background queue (e.g. the callback is marked @Sendable and hops via Task { @MainActor in ... } or equivalent), and the answer explains why this matters under Swift 6
