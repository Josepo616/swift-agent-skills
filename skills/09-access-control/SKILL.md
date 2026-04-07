---
description: Apply Swift access control levels and code organization
user_invocable: true
---

# Skill 09 — Access Control & Code Organization

**Source:** Swift.org, API Design Guidelines  
**Applies to:** All Swift files

---

## Access Level Hierarchy

```
open        → Subclass/override from any module (classes only)
public      → Use from any module, no subclass
package     → Visible within the same Swift package (5.9+)
internal    → Visible within the same module (DEFAULT)
fileprivate → Visible within the same file
private     → Visible within the same declaration + extensions in same file
```

## Default to Most Restrictive

```swift
// GOOD: Only expose what's needed
@Observable
final class ItemListViewModel {
    var searchText: String = ""              // internal — views need this
    var isLoading: Bool = false              // internal — views read this
    private(set) var items: [Item] = []      // read-only externally

    private let service: DataFetching        // private — implementation detail

    func fetchItems() async { ... }          // internal — views call this
    private func parseResponse(_ data: Data) throws -> Item { ... }
}
```

## File Organization Within a Swift File

```swift
// 1. Type definition
struct ItemDetailView: View {
    let item: Item
    var body: some View { ... }
}

// 2. Protocol conformances in extensions
// MARK: - Equatable
extension ItemDetailView: Equatable {
    static func == (lhs: Self, rhs: Self) -> Bool {
        lhs.item.id == rhs.item.id
    }
}

// 3. Private helpers in private extensions
// MARK: - Private Helpers
private extension ItemDetailView {
    func formattedDate() -> String { ... }
}
```

## Formatting Conventions

- [ ] Prefer `guard` for early returns over nested `if`
- [ ] Use trailing closures for the last (or only) closure parameter
- [ ] Prefer `let` over `var` wherever possible
- [ ] Use type inference — specify types only when needed for clarity
- [ ] Use `// MARK: -` to organize sections within a file

## Key Rules

- [ ] Start `private`, widen access only when needed
- [ ] Use `private(set)` for properties that should be read-only externally
- [ ] Use `fileprivate` only when sharing across types in the same file
- [ ] Use `package` (Swift 5.9+) for cross-module but within-package visibility
- [ ] One type per file (small related types can share a file)

## Mistakes to Avoid

- [ ] Making everything `public` or `internal` by default
- [ ] Using force unwrapping (`!`) outside of tests
- [ ] Missing `// MARK:` sections in files over ~50 lines
- [ ] Overusing `self.` when not needed for disambiguation
