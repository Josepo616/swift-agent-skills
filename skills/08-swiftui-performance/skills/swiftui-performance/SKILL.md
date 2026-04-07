---
description: Optimize SwiftUI view performance
user_invocable: true
---

# Skill 08 — SwiftUI Performance Optimization

**Source:** Apple WWDC 2025, SwiftUI Documentation  
**Applies to:** All views, lists, animations

---

## 1. Break Views into Small Components

```swift
// BAD: Monolithic view — any state change re-evaluates everything
struct BadView: View {
    @State var name = ""
    @State var items: [Item] = []

    var body: some View {
        VStack {
            TextField("Name", text: $name)
            ForEach(items) { ExpensiveRow(item: $0) }
        }
    }
}

// GOOD: Separate sub-views so SwiftUI diffs independently
struct GoodView: View {
    var body: some View {
        VStack {
            NameInput()
            ItemList()
        }
    }
}
```

## 2. Use Lazy Containers for Large Lists

```swift
// BAD: Creates ALL child views upfront
ScrollView { VStack { ForEach(items) { ... } } }

// GOOD: Only creates visible views
ScrollView { LazyVStack { ForEach(items) { ... } } }
```

## 3. No Expensive Work in `body`

```swift
// BAD: Formatter created every render
var body: some View {
    Text(date, formatter: DateFormatter())  // new instance each time
}

// GOOD: Cached formatter
private static let formatter: DateFormatter = { ... }()
var body: some View {
    Text(date, formatter: Self.formatter)
}
```

## 4. Stable View Identity

```swift
// BAD: New UUID each render = full teardown/rebuild
ForEach(items) { item in
    ItemView(item: item).id(UUID())   // NEVER do this
}

// GOOD: Stable identity
ForEach(items) { item in
    ItemView(item: item).id(item.id)
}
```

## 5. Use `.task` Over `.onAppear`

```swift
// Auto-cancels when view disappears
.task { await viewModel.loadData() }

// Re-runs when dependency changes
.task(id: selectedCategory) {
    await viewModel.loadItems(for: selectedCategory)
}
```

## 6. Equatable Views for Complex Components

```swift
struct ExpensiveView: View, Equatable {
    let data: SomeData

    static func == (lhs: Self, rhs: Self) -> Bool {
        lhs.data.id == rhs.data.id
    }

    var body: some View { /* complex rendering */ }
}
```

## Debugging Tools

- `Self._printChanges()` in view body — shows what triggered re-render
- Instruments > SwiftUI profiling template
- Xcode Previews for visual regression

## Key Rules

- [ ] Break large views into small, focused sub-views
- [ ] Use `LazyVStack`/`LazyHStack` inside `ScrollView` for lists
- [ ] Cache formatters and expensive computed values as `static let`
- [ ] Use stable `.id()` values — never `UUID()` or `Date()`
- [ ] Use `.task { }` for async work, not `.onAppear`

## Mistakes to Avoid

- [ ] One monolithic view that re-renders on every state change
- [ ] `VStack` + `ForEach` for hundreds of items (use Lazy)
- [ ] Filtering/sorting arrays inside `body`
- [ ] `.id(UUID())` or `.id(Date())` destroying view identity
