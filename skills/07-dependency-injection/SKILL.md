---
description: Implement dependency injection patterns in SwiftUI
user_invocable: true
---

# Skill 07 — Dependency Injection in SwiftUI

**Source:** Apple Documentation, Community Patterns  
**Applies to:** ViewModels, services, testing

---

## Pattern 1: Constructor Injection (Recommended Default)

```swift
@Observable
final class ItemListViewModel {
    private let service: DataFetching

    // Default provides production instance; tests pass a mock
    init(service: DataFetching = MockDataService()) {
        self.service = service
    }
}

// Production
@State var viewModel = ItemListViewModel()

// Tests
let viewModel = ItemListViewModel(service: MockDataService())
```

## Pattern 2: SwiftUI Environment (for view-layer dependencies)

```swift
// Modern approach with @Entry macro (Xcode 16+)
extension EnvironmentValues {
    @Entry var theme: Theme = .default
}

// Inject at parent
ContentView()
    .environment(\.theme, .dark)

// Consume in child
struct ContentView: View {
    @Environment(\.theme) var theme
}
```

## Pattern 3: @Environment with @Observable (iOS 17+)

```swift
@Observable
final class AuthManager {
    var isAuthenticated = false
}

// At app root
@main
struct MyApp: App {
    @State private var authManager = AuthManager()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(authManager)
        }
    }
}

// In any child view
struct ProfileView: View {
    @Environment(AuthManager.self) var authManager
}
```

## When to Use Each

| Pattern | Use When |
|---------|----------|
| Constructor injection | ViewModels, services, non-view code |
| `@Environment` (values) | Theme, locale, feature flags |
| `@Environment` (Observable) | Shared managers (Auth, Settings) |

## Key Rules

- [ ] Default to constructor injection for ViewModels and services
- [ ] Use protocols for injected dependencies to enable testing
- [ ] Use `@Environment` for truly app-wide shared state
- [ ] Provide default parameter values for production instances

## Mistakes to Avoid

- [ ] Using `@EnvironmentObject` without providing it (runtime crash, no compile check)
- [ ] Overusing environment for everything — makes dependencies implicit
- [ ] Not using protocols for injected dependencies (hard to test)
- [ ] Passing dependencies through 5+ layers manually when Environment is cleaner
