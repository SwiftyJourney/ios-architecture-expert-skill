---
type: llm
focus: trace
weight: 1
---

The selection closure is wired exactly once, from the root; if it captures the scene delegate/composer strongly, that is acceptable for a scene-lifetime owner, but it must not create a cycle with a shorter-lived object (e.g. a view controller capturing itself or its presenter)
