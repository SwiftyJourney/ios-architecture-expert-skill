---
type: llm
focus: trace
weight: 1
---

The answer identifies that FeedViewAdapter has no other owner, so WeakRefVirtualProxy(viewAdapter) lets it deallocate immediately and feed updates go nowhere -- that is why the screen is blank
