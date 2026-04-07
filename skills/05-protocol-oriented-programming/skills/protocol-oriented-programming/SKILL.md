---
description: Apply protocol-oriented programming patterns in Swift
user_invocable: true
---

# Skill 05 — Protocol-Oriented Programming

**Source:** Swift.org, WWDC, Community Best Practices  
**Applies to:** Service layer, dependency injection, abstraction boundaries

---

## Core Idea

Swift favors **protocol composition over class inheritance**. Define small, focused protocols and compose them.

## Protocol Definition

```swift
// Small, focused protocols (Interface Segregation Principle)
protocol DataFetching: Sendable {
    func fetchItem(by id: String) async throws -> Item
}

protocol DataPersisting {
    func save(_ item: Item) throws
    func fetchAll() throws -> [Item]
}

// Compose when needed
typealias DataService = DataFetching & DataPersisting
```

## Default Implementations via Extensions

```swift
protocol Loadable {
    associatedtype Content
    var isLoading: Bool { get set }
    func load() async throws -> Content
}

extension Loadable {
    mutating func performLoad() async {
        isLoading = true
        defer { isLoading = false }
        // shared loading logic
    }
}
```

## Constrained Extensions

```swift
extension Collection where Element: Item {
    var completedCount: Int {
        filter { $0.status == .completed }.count
    }
}
```

## The `any` Keyword (Existential Types)

```swift
// Explicit existential marking (Swift 5.7+)
func process(services: [any DataFetching]) { ... }

// Generic constraint (PREFERRED for performance)
func process<T: DataFetching>(service: T) { ... }

// Use `any` for heterogeneous collections
// Use generics when all items are the same concrete type
```

## When to Use Protocols vs Concrete Types

| Use Protocol When | Use Concrete Type When |
|---|---|
| Multiple conformances exist or are planned | Only one implementation ever |
| You need testability via mocking | Internal helper with no testing need |
| Abstraction boundary between layers | Simple data container |

## Key Rules

- [ ] Keep protocols small and focused — one responsibility each
- [ ] Prefer composition (`Identifiable & Codable`) over deep protocol hierarchies
- [ ] Use generics over `any` (existential) for performance-critical paths
- [ ] Add default implementations in extensions for shared behavior
- [ ] Only create protocols when there's a real need — avoid "single-implementation protocol" antipattern

## Mistakes to Avoid

- [ ] Creating a protocol for every type (protocol pollution)
- [ ] Forgetting default implementations use **static dispatch** (not dynamic)
- [ ] Deep protocol hierarchies mirroring class inheritance
- [ ] Using `any ProtocolWithAssociatedType` when a generic is simpler
