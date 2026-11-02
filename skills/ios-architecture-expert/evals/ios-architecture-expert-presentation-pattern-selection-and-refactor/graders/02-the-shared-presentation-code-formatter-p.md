---
type: llm
focus: trace
weight: 1
---

The shared presentation code (formatter/presenter/view model core) has no 'import UIKit'/'import SwiftUI'; the 'Member since' string is produced there, and avatar bytes cross the boundary as Data or via an injected transformer, never as UIImage
