---
type: llm
focus: trace
weight: 1
---

If NWPathMonitor-driven retry is offered, it fires only on a transition from unsatisfied to satisfied (not on the monitor's first 'satisfied' report), so a server outage with network available cannot cause a retry loop
