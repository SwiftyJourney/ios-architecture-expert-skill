---
type: llm
focus: trace
weight: 1
---

It traces the actual ownership cycle (FeedViewController -> delegate/FeedPresentationAdapter -> presenter -> view references back to FeedViewController) and says only the reference back to the controller needs to be weak
