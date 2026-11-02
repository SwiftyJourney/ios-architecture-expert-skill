---
type: llm
focus: trace
weight: 1
---

The fixed composer keeps feedView pointing at the FeedViewAdapter strongly while loadingView/errorView use WeakRefVirtualProxy(controller)
