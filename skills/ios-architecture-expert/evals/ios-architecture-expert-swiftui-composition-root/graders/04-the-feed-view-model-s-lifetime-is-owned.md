---
type: llm
focus: trace
weight: 1
---

The feed view model outlives the feed view (owned by the root/composer) AND the reload is actually prevented: the feed view's `.task` calls a first-load guard (e.g. `loadIfNeeded` / a `hasLoaded` check), so popping back from comments does not refetch, while refresh/retry call the unguarded load
