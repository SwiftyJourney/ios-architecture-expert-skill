---
tags: [ios-architecture-expert]
max_turns: 40
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

Our codebase has `AnalyticsManager.shared` (a `static let` with a private init), `APIClient.shared` (a `static var` so tests can swap it), and `URLSession.shared` used directly inside three view controllers. A reviewer just told me "singletons are bad, remove them all". Is that right? What should I actually do here, and in what order? Show the refactor for the API client.
