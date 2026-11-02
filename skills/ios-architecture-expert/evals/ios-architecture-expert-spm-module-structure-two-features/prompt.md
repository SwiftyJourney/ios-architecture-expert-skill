---
tags: [ios-architecture-expert]
max_turns: 40
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

Our app has two features (Feed and ImageComments) built clean-architecture style. We're moving from one Xcode target with folders to a modular SPM setup. Propose the package layout: which targets, which products, what depends on what, where CoreData and the localized presentation strings live, where shared code (HTTPClient, the generic presenter) goes, and how the Xcode app target consumes it all. Include the Package.swift.
