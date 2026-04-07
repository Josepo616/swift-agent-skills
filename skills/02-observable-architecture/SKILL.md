---
description: Use @Observable and modern MVVM architecture patterns in SwiftUI
user_invocable: true
---

# Skill 02 — @Observable & Modern MVVM Architecture

**Source:** Apple WWDC 2024-2025, SwiftUI Documentation  
**Applies to:** ViewModels, state management, data flow

---

## Use @Observable (iOS 17+), Not ObservableObject

`@Observable` is the recommended approach for all new projects. It provides **per-property tracking** — SwiftUI only re-renders views that read the specific property that changed.

### Before (Old Way)
```swift
class ViewModel: ObservableObject {
    @Published var name: String = ""
    @Published var isLoading: Bool = false
}

struct MyView: View {
    @StateObject var viewModel = ViewModel()
}
```

### After (Modern Way)
```swift
@Observable
final class ViewModel {
    var name: String = ""
    var isLoading: Bool = false
}

struct MyView: View {
    @State var viewModel = ViewModel()
}
```

## Property Wrapper Migration

| Old | New |
|-----|-----|
| `@StateObject` | `@State` |
| `@ObservedObject` | Direct reference (no wrapper) |
| `@EnvironmentObject` | `@Environment` |
| `@Published` | Removed — all `var` are tracked |
| N/A | `@Bindable` (for creating bindings) |

## MVVM Structure

```swift
@Observable
@MainActor
final class ItemListViewModel {
    var searchText: String = ""
    var isLoading: Bool = false
    var selectedItem: Item?
    var errorMessage: String?

    private let service: DataFetching

    init(service: DataFetching = MockDataService()) {
        self.service = service
    }

    func fetchItems() async { ... }
}
```

## Key Rules

- [ ] Use `@State var viewModel = MyViewModel()` to own the ViewModel in a view
- [ ] Use `@Bindable` when creating bindings from `@Observable` properties
- [ ] Keep ViewModels importing only `Foundation`, not `SwiftUI` — improves testability
- [ ] Mark ViewModels as `@MainActor` when they do async work updating UI state
- [ ] Use `.task { }` modifier for async work — it auto-cancels on view disappearance

## Mistakes to Avoid

- [ ] Using `@ObservedObject` where `@State` should be used (causes ViewModel recreation)
- [ ] Creating ViewModels for trivially simple views that don't need them
- [ ] Importing `SwiftUI` in ViewModels
- [ ] Using `let` instead of `@State var` for owned ViewModels
