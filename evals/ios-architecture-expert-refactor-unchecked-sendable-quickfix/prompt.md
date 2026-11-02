---
tags: [ios-architecture-expert]
max_turns: 40
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

Migrating to Swift 6 strict concurrency and this file lights up with warnings. Can you make it concurrency-safe? Quick fix preferred, we ship Friday. Refactor it and write the resulting file(s) to the output directory.

```swift
import UIKit

final class FavoritesViewModel: ObservableObject, @unchecked Sendable {
    static let shared = FavoritesViewModel()

    @Published var favorites: [FavoriteCellModel] = []
    @Published var isLoading = false

    private var lastFetch: Date?

    func load() {
        if let lastFetch, Date().timeIntervalSince(lastFetch) < 300 { return }
        isLoading = true
        APIClient.shared.get("/v1/favorites") { [weak self] data in
            DispatchQueue.main.async {
                guard let self, let data else { return }
                let items = try? JSONDecoder().decode([FavoriteDTO].self, from: data)
                self.favorites = (items ?? []).map {
                    FavoriteCellModel(title: $0.title, image: UIImage(data: $0.imageData) ?? UIImage())
                }
                self.lastFetch = Date()
                self.isLoading = false
            }
        }
    }
}

final class FavoritesViewController: UITableViewController {
    let viewModel = FavoritesViewModel.shared

    func didSelect(_ item: FavoriteCellModel) {
        navigationController?.pushViewController(FavoriteDetailViewController(item: item), animated: true)
    }
}
```
