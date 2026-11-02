---
type: llm
focus: trace
weight: 1
---

The refactor is presented as incremental steps where each step keeps existing tests compiling/passing (e.g. inject the same instance first, then narrow the dependency, then move creation to the root) -- not a one-shot rewrite that breaks call sites
