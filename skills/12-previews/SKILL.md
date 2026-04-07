---
description: Set up and use SwiftUI previews effectively
user_invocable: true
---

# Skill 12 — SwiftUI Preview Best Practices

**Source:** Apple Documentation, Community  
**Applies to:** All views during development

---

## Multiple Preview States

```swift
#Preview("Default State") {
    ContentView()
        .environment(ItemListViewModel())
}

#Preview("Loading State") {
    ContentView()
        .environment(ItemListViewModel.loadingPreview())
}

#Preview("With Data") {
    ContentView()
        .environment(ItemListViewModel.populatedPreview())
}

#Preview("Error State") {
    ContentView()
        .environment(ItemListViewModel.errorPreview())
}
```

## Preview Data

```swift
// Preview Content/PreviewData.swift
extension Item {
    static let previewActive = Item(
        id: UUID(),
        title: "Sample Task",
        status: .active,
        priority: 3,
        description: "A sample item for previews",
        createdAt: .now
    )

    static let previewCompleted = Item(
        id: UUID(),
        title: "Finished Task",
        status: .completed,
        priority: 1,
        description: "A completed item",
        createdAt: .now.addingTimeInterval(-3600)
    )

    static let previewSamples: [Item] = [
        .previewActive, .previewCompleted, /* ... */
    ]
}
```

## Accessibility Previews

```swift
#Preview("Large Dynamic Type") {
    ItemDetailView(item: .previewActive)
        .environment(\.sizeCategory, .accessibilityExtraExtraExtraLarge)
}

#Preview("Dark Mode") {
    ItemDetailView(item: .previewActive)
        .preferredColorScheme(.dark)
}
```

## Key Rules

- [ ] Create previews for every meaningful state (default, loading, error, populated)
- [ ] Create shared preview data in `Preview Content/` folder
- [ ] Use `static let` preview instances on model types
- [ ] Test Dynamic Type and Dark Mode in previews
- [ ] Keep preview data realistic but simple
