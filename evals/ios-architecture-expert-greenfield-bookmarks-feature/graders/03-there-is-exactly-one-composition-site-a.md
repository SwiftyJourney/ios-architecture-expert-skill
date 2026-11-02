---
type: llm
focus: trace
weight: 1
---

There is exactly one composition site (a composer/static factory/service in the app target, possibly split across files of that target) that instantiates every concrete dependency; no feature, presentation, or UI type constructs its own loader, client, store, or presenter internally
