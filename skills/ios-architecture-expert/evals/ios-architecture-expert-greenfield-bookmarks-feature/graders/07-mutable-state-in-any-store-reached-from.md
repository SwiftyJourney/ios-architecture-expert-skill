---
type: llm
focus: trace
weight: 1
---

Mutable state in any store reached from async code is protected by actor isolation (an actor or @MainActor), not by a lock or Mutex held in async code
