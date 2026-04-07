---
description: Apply Swift concurrency patterns with async/await, Actors, and Sendable
user_invocable: true
---

# Skill 03 — Swift Concurrency (async/await, Actors, Sendable)

**Source:** Apple WWDC 2024-2025, Swift.org Concurrency Migration Guide  
**Applies to:** Networking, service calls, any asynchronous work

---

## async/await Fundamentals

```swift
func fetchItem(by id: String) async throws -> Item {
    let (data, response) = try await URLSession.shared.data(for: request)
    guard let http = response as? HTTPURLResponse, http.statusCode == 200 else {
        throw NetworkError.invalidResponse
    }
    return try JSONDecoder().decode(Item.self, from: data)
}
```

## Structured Concurrency with Task Groups

```swift
func fetchMultiple(ids: [Int]) async throws -> [User] {
    try await withThrowingTaskGroup(of: User.self) { group in
        for id in ids {
            group.addTask { try await fetchUser(id: id) }
        }
        return try await group.reduce(into: []) { $0.append($1) }
    }
}
```

Child tasks auto-cancel when the parent scope exits or a sibling throws.

## @MainActor for UI Work

```swift
@MainActor
@Observable
final class ContentViewModel {
    var items: [Item] = []

    func loadItems() async {
        let fetched = await repository.fetchItems() // hops off main
        items = fetched // back on main, safe for UI
    }
}
```

## Actors for Thread-Safe State

```swift
actor ImageCache {
    private var cache: [URL: Image] = [:]

    func image(for url: URL) -> Image? { cache[url] }
    func setImage(_ image: Image, for url: URL) { cache[url] = image }
}
```

## Sendable Compliance

- Value types (structs, enums) are implicitly `Sendable` if all stored properties are `Sendable`
- `final class` is `Sendable` if all stored properties are `let` and `Sendable`
- Actors are automatically `Sendable`
- Use `@unchecked Sendable` only as a **last resort**

## Swift 6 Strict Concurrency

- Compile-time data race detection
- Global variables must be `let` constants or isolated to an actor
- Enable `StrictConcurrency` warning level incrementally

## Key Rules

- [ ] Prefer structured concurrency (`.task { }`, `TaskGroup`) over unstructured `Task { }`
- [ ] Use `@MainActor` on ViewModels, not on every function individually
- [ ] Handle cancellation: check `Task.isCancelled` or call `try Task.checkCancellation()`
- [ ] Use `.task { }` modifier instead of `.onAppear` for async work
- [ ] Use `.task(id:)` to re-run async work when a dependency changes

## Mistakes to Avoid

- [ ] Using `Task { }` when `.task { }` modifier would auto-cancel properly
- [ ] Forgetting `Task.detached` doesn't inherit actor context
- [ ] Making everything `@MainActor` — isolate only what needs UI access
- [ ] Using `@unchecked Sendable` to silence warnings without real thread safety
