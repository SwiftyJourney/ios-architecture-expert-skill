---
tags: [ios-architecture-expert]
max_turns: 40
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

We have a generic `ListViewController` (UITableViewDiffableDataSource of `CellController`s) shared by the Feed and Comments screens. Product wants infinite scroll on the Feed only. My plan: add `var supportsPagination = false` and `var onLoadMore: (() -> Void)?` to ListViewController, detect when the user scrolls near the bottom, call onLoadMore, and when the next page arrives append the new items to the snapshot, skipping ids we already have. The feed API is keyset-paginated (`?after_id=`). We also cache the feed offline. Write the design and the code.
