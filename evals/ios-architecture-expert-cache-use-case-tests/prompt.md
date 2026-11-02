---
tags: [ios-architecture-expert]
max_turns: 40
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

hey, I need tests for a save/cache use case before I write it (TDD-ish). The plan: a LocalGalleryLoader that takes a GalleryStore dependency and a way to get the current date. save(_ items: [GalleryItem]) async throws should first delete the old cache, and only if deletion succeeds, insert the new items with the current timestamp. If deletion fails it should rethrow and never insert. Can you write the test suite (and whatever minimal production code/protocols the tests need to compile)? Put the files in the output directory.
