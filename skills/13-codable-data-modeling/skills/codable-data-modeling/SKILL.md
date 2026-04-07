---
description: Design data models with Codable and JSON parsing
user_invocable: true
---

# Skill 13 — Codable & Data Modeling

**Source:** Swift.org, Apple Documentation  
**Applies to:** Models, JSON parsing, API responses

---

## Model Design

```swift
struct Item: Identifiable, Codable, Sendable {
    let id: UUID
    let title: String
    let category: Category
    let priority: Int
    let description: String
    let createdAt: Date

    init(
        id: UUID = UUID(),
        title: String,
        category: Category,
        priority: Int,
        description: String,
        createdAt: Date = .now
    ) {
        self.id = id
        self.title = title
        self.category = category
        self.priority = priority
        self.description = description
        self.createdAt = createdAt
    }
}
```

## Enum with Raw Values

```swift
enum Category: String, CaseIterable, Codable, Sendable {
    case work, personal, health, finance, education, other
}
```

## Decoding API Responses

```swift
// When API JSON keys differ from Swift property names
struct APIResponse: Codable {
    let status: String
    let priority: Int
    let description: String

    enum CodingKeys: String, CodingKey {
        case status
        case priority
        case description = "item_description"  // map different key
    }
}
```

## Safe Decoding Patterns

```swift
// Decode with custom date strategy
let decoder = JSONDecoder()
decoder.dateDecodingStrategy = .iso8601
decoder.keyDecodingStrategy = .convertFromSnakeCase

let item = try decoder.decode(Item.self, from: data)
```

## Key Rules

- [ ] Conform models to `Identifiable`, `Codable`, `Sendable`
- [ ] Use `let` for properties that shouldn't change after creation
- [ ] Provide default values in `init` for convenience (`id: UUID = UUID()`)
- [ ] Use `CodingKeys` only when API keys differ from property names
- [ ] Use `String` raw values for enums that need JSON/API compatibility

## Mistakes to Avoid

- [ ] Using classes for simple data models (prefer structs)
- [ ] Forgetting `Sendable` conformance for types crossing concurrency boundaries
- [ ] Manual `encode`/`decode` when auto-synthesis works
- [ ] Mutable (`var`) properties that should be immutable (`let`)
