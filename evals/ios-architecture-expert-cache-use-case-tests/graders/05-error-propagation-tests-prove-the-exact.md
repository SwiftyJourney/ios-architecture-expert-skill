---
type: llm
focus: trace
weight: 1
---

Error-propagation tests assert the specific stubbed error is rethrown, compared by value (e.g. `as NSError` domain + code, which survives errors being recreated across boundaries), not merely that some error was thrown
