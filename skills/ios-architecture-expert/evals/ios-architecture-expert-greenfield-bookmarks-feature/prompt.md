---
tags: [ios-architecture-expert]
max_turns: 40
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

I'm starting a new feature in my iOS app: 'Bookmarks'. Users see a list of bookmarked articles (title, author, thumbnail image) loaded from our REST API (GET /v1/bookmarks), with offline support so the last loaded list shows when there's no connection. We use Swift 6 and SPM. Design and write the Swift code for this feature end to end -- models, networking, caching, presentation, a UIKit list screen, and the code that wires it all together at app launch. Write the source files to the output directory, organized into folders the way you'd organize the modules.
