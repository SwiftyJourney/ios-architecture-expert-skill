---
tags: [ios-architecture-expert]
max_turns: 40
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

New SwiftUI app (iOS 26, Swift 6). I have Feed and Comments modules from our shared package, with a FeedService actor-ish orchestrator already written. There's no SceneDelegate anymore -- where do I wire everything up, how do views get their dependencies, and who owns navigation? Right now each View creates its own @State view model and pushes destinations itself. Write the composition code.
