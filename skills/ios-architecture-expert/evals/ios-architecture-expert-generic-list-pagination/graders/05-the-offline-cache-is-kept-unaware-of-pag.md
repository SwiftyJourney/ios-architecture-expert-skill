---
type: llm
focus: trace
weight: 1
---

The offline cache is kept unaware of pagination (saves/loads the accumulated items through its existing API), AND the page builder keeps the accumulated items in memory by default; re-reading the cache on every load-more to rebuild the list is at most an optional optimisation whose expiry/failed-save risk is stated
