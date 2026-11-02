---
tags: [ios-architecture-expert]
max_turns: 40
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

Our ProfileViewController is a mess and my lead says we should 'move to MVVM'. It's ~400 lines: it calls ProfileService.shared.fetch(), formats the join date into 'Member since March 2023', converts the avatar Data into a UIImage, shows/hides a spinner and an error label, and pushes SettingsViewController when you tap the gear. We're also starting a watchOS app that shows the same profile info. Which pattern should we actually use and why? Refactor it and write the resulting files to the output directory.
