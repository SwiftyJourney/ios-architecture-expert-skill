---
type: llm
focus: trace
weight: 1
---

The offline behaviour is implemented as a composition: remote first, successful results saved to the cache, cached list shown when the remote fails, and an error (not a silent empty list) when the remote fails and nothing was ever cached
