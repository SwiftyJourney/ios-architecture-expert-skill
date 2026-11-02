---
tags: [ios-architecture-expert]
max_turns: 40
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

Before I load the feed I want to check whether the user has internet, and show an 'offline' screen if not, so I don't waste a request. I was going to use NWPathMonitor and cache the status in a shared ConnectivityService that the view model reads. Write the code for this.
