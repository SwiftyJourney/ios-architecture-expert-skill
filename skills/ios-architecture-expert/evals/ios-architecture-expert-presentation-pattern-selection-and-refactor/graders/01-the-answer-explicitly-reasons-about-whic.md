---
type: llm
focus: trace
weight: 1
---

The answer chooses the pattern by client: because the watch client is SwiftUI, it shares a Foundation-only presenter/mapper and gives each SwiftUI client an @Observable view model; it does NOT build MVP view protocols + adapter + WeakRefVirtualProxy as the shared layer, and it does not justify the choice from an acronym/criteria table alone
