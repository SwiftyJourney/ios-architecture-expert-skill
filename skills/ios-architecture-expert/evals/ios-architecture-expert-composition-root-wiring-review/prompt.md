---
tags: [ios-architecture-expert]
max_turns: 40
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

Review this SceneDelegate: it creates URLSessionHTTPClient in three places, builds a new CoreDataFeedStore per screen, calls FeedUIComposer from inside FeedViewController when navigating to comments, and crashes at startup if CoreData fails to load. What's wrong and how should it be structured?
