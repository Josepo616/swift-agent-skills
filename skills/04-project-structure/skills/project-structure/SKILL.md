---
description: Organize SwiftUI project structure and file layout
user_invocable: true
---

# Skill 04 — SwiftUI Project Structure & File Organization

**Source:** Apple Sample Apps, Community Best Practices  
**Applies to:** File and folder layout, group structure

---

## Principle: Feature-Based Over Layer-Based

Group code by **feature first**, then by role within the feature. This keeps related code together and makes features self-contained.

## Recommended Structure for Small-to-Medium Apps

```
MyApp/
├── MyAppApp.swift                    // @main entry point
├── ContentView.swift                 // Root navigation
├── Assets.xcassets/
├── Models/
│   ├── Item.swift
│   └── Category.swift
├── Services/
│   ├── DataFetching.swift            // Protocol
│   ├── MockDataService.swift
│   └── APIDataService.swift
├── ViewModels/
│   └── ItemListViewModel.swift
├── Views/
│   ├── MainTabView.swift
│   ├── ItemDetailView.swift
│   ├── ItemRow.swift
│   ├── HistoryView.swift
│   └── SettingsView.swift
├── Extensions/
│   ├── Color+Theme.swift
│   └── Date+Formatting.swift
└── Preview Content/
    └── PreviewData.swift
```

## For Larger Projects — Feature-Based

```
Features/
├── ItemList/
│   ├── ItemListView.swift
│   ├── ItemListViewModel.swift
│   └── Components/
│       └── ItemCard.swift
├── Detail/
│   ├── DetailView.swift
│   └── Components/
│       └── DetailHeader.swift
└── Settings/
    ├── SettingsView.swift
    └── Components/
```

## Key Rules

- [ ] Group by feature first, then by role within the feature
- [ ] Keep feature folders self-contained: views, viewmodel, and components together
- [ ] Shared/reusable components go in a `Shared/` or `Views/` directory
- [ ] Models and services that span features get their own top-level folders
- [ ] Match Xcode group structure to filesystem structure
- [ ] One type per file (with small closely-related types as exceptions)

## Mistakes to Avoid

- [ ] Putting all views in one folder, all models in another (unmanageable at scale)
- [ ] Creating deeply nested folder hierarchies for a small project
- [ ] Mismatching Xcode groups and filesystem folders
- [ ] Dumping everything in the root directory
