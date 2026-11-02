---
tags: [ios-architecture-expert]
max_turns: 40
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

After our 'fix memory leaks' PR the feed screen is blank — no cells, no error, no spinner. The PR wrapped every reference the presenter holds in our WeakRefVirtualProxy, because a reviewer said presenters should never hold views strongly. Here's the composer now:

```swift
public enum FeedUIComposer {
    @MainActor
    public static func feedComposedWith(loader: @escaping () async throws -> [FeedImage]) -> FeedViewController {
        let adapter = FeedPresentationAdapter(loader: loader)
        let controller = FeedViewController(delegate: adapter)
        let viewAdapter = FeedViewAdapter(controller: controller, imageLoader: imageLoader)
        adapter.presenter = FeedPresenter(
            feedView: WeakRefVirtualProxy(viewAdapter),
            loadingView: WeakRefVirtualProxy(controller),
            errorView: WeakRefVirtualProxy(controller))
        return controller
    }
}
```

FeedViewController holds `delegate` strongly, FeedPresentationAdapter holds `presenter` strongly, FeedViewAdapter holds `controller` weakly. What's wrong and what's the right rule?
