---
description: Persistence with SwiftData: @Model, @Query, #Predicate, relationships, migrations, and iCloud sync
user_invocable: true
---

# Skill 14 — SwiftData

**Source:** Apple WWDC 2023–2025, Swift.org  
**Applies to:** Persistence, data modeling, iCloud sync  
**Minimum target:** iOS 17+ (features noted where iOS 18+ or iOS 26+ required)

---

## @Model Macro and Class Design

SwiftData models are **classes**, not structs. Apply `@Model` to opt in.

```swift
import SwiftData

@Model
final class Trip {
    var name: String
    var destination: String
    var startDate: Date
    var endDate: Date
    var budget: Decimal

    @Attribute(.externalStorage) var photo: Data?
    @Transient var isEditing: Bool = false

    init(name: String, destination: String, startDate: Date, endDate: Date, budget: Decimal) {
        self.name = name
        self.destination = destination
        self.startDate = startDate
        self.endDate = endDate
        self.budget = budget
    }
}
```

### @Attribute Options

| Option | Purpose |
|---|---|
| `.unique` | Upsert on conflict (iOS 17+). **Breaks CloudKit — never use with iCloud sync.** |
| `.externalStorage` | Store large blobs outside the database. Only for `Data`. It is a hint, not a guarantee. |
| `.preserveValueOnDeletion` | Keep value in history tombstone after deletion (iOS 18+). |
| `.spotlight` | Index property for Spotlight search. |
| `.allowsCloudEncryption` | Encrypt value in CloudKit. |
| `originalName:` | Rename property without migration (maps to old column name). |

### @Transient

`@Transient` properties are **not persisted**. They reset to their **default value** on every fetch — they do not retain in-memory values across fetches. If you need a derived value, prefer a computed property instead.

### #Unique Macro (iOS 18+)

Declares uniqueness constraints. **Only one `#Unique` per model** — combine multiple constraints in a single macro call.

```swift
// CORRECT: one #Unique with multiple constraint groups
@Model
final class Recipe {
    #Unique<Recipe>([\.name, \.category], [\.externalID])

    var name: String
    var category: String
    var externalID: String
    // ...
}

// WRONG: multiple #Unique macros — only the last one is applied
@Model
final class Recipe {
    #Unique<Recipe>([\.name, \.category])
    #Unique<Recipe>([\.externalID])  // silently overwrites the first
    // ...
}
```

### Enums in Models

Enum properties must conform to `Codable`. **Enums with associated values ARE supported** — LLMs frequently claim otherwise.

```swift
// This works fine
enum Status: Codable {
    case draft
    case published(date: Date)
    case archived(reason: String)
}

@Model
final class Article {
    var title: String
    var status: Status  // associated values are supported
    // ...
}
```

---

## @Query Macro

`@Query` fetches and observes model data **inside SwiftUI views only**. It does not work in ViewModels, services, or any non-view context.

```swift
struct TripListView: View {
    @Query(sort: \.startDate, order: .reverse)
    private var trips: [Trip]

    var body: some View {
        List(trips) { trip in
            TripRow(trip: trip)
        }
    }
}
```

### Sorting and Filtering

```swift
// Multiple sort descriptors
@Query(
    filter: #Predicate<Trip> { $0.budget > 1000 },
    sort: [
        SortDescriptor(\.startDate, order: .reverse),
        SortDescriptor(\.name)
    ]
)
private var expensiveTrips: [Trip]
```

### Dynamic Queries

Use `init` to configure `@Query` dynamically based on state.

```swift
struct TripListView: View {
    @Query private var trips: [Trip]

    init(searchText: String, sortOrder: SortDescriptor<Trip>) {
        _trips = Query(
            filter: #Predicate<Trip> { trip in
                searchText.isEmpty || trip.name.localizedStandardContains(searchText)
            },
            sort: [sortOrder]
        )
    }

    var body: some View {
        List(trips) { trip in
            TripRow(trip: trip)
        }
    }
}
```

### @Query with Animation

```swift
@Query(sort: \.startDate, animation: .default)
private var trips: [Trip]
```

Changes to results automatically animate the view transition.

---

## #Predicate Macro

`#Predicate` is a **compile-time macro**, not `NSPredicate`. It uses Swift syntax but has significant runtime limitations — many expressions that compile will **crash at runtime**.

### Safe Operations

```swift
// String matching — use localizedStandardContains, NOT lowercased().contains()
#Predicate<Trip> { $0.name.localizedStandardContains(searchText) }

// Comparison operators
#Predicate<Trip> { $0.budget > 1000 && $0.startDate > Date.now }

// Optional checks
#Predicate<Trip> { $0.photo != nil }

// Collection check — use !isEmpty, NOT .isEmpty == false
#Predicate<Trip> { !$0.activities.isEmpty }

// starts(with:) — NOT hasPrefix()
#Predicate<Trip> { $0.name.starts(with: "A") }
```

### Predicate Crash Catalog

These expressions **compile but crash at runtime**:

```swift
// CRASHES: .isEmpty == false (use !.isEmpty instead)
#Predicate<Trip> { $0.activities.isEmpty == false }

// CRASHES: hasPrefix / hasSuffix (not supported)
#Predicate<Trip> { $0.name.hasPrefix("A") }

// CRASHES: lowercased(), uppercased()
#Predicate<Trip> { $0.name.lowercased().contains("trip") }

// CRASHES: map, reduce, count(where:), first, filter on collections
#Predicate<Trip> { $0.activities.count(where: { $0.isOutdoor }) > 0 }

// CRASHES: computed properties
#Predicate<Trip> { $0.durationInDays > 7 }  // durationInDays is computed

// CRASHES: @Transient properties
#Predicate<Trip> { $0.isEditing }  // isEditing is @Transient

// CRASHES: custom Codable structs used in comparison
#Predicate<Trip> { $0.address.city == "Paris" }  // address is a custom Codable struct

// CRASHES: regex patterns
#Predicate<Trip> { $0.name.contains(/^trip/i) }

// CRASHES: custom operators
#Predicate<Trip> { $0.name ~= pattern }
```

### Reusable Predicates

```swift
extension Trip {
    static func upcoming(after date: Date = .now) -> Predicate<Trip> {
        #Predicate<Trip> { $0.startDate > date }
    }
}

// Usage
@Query(filter: Trip.upcoming())
private var upcomingTrips: [Trip]
```

---

## ModelContainer and ModelContext Lifecycle

### ModelContainer Setup

Create one container at app launch and inject it into the environment.

```swift
@main
struct TripApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: [Trip.self, Activity.self])
    }
}
```

For custom configuration:

```swift
@main
struct TripApp: App {
    let container: ModelContainer

    init() {
        let schema = Schema([Trip.self, Activity.self])
        let config = ModelConfiguration(
            "TripStore",
            schema: schema,
            isStoredInMemoryOnly: false,
            allowsSave: true
        )
        container = try! ModelContainer(for: schema, configurations: config)
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(container)
    }
}
```

### ModelContext Rules

- **`mainContext`** — auto-provided, runs on `@MainActor`, has **autosave enabled by default**
- **Custom contexts** — created manually, do **not** autosave by default
- Autosave timing is unpredictable — **prefer explicit `try context.save()`** for important writes

```swift
// In a SwiftUI view — uses mainContext from environment
@Environment(\.modelContext) private var context

func addTrip() {
    let trip = Trip(name: "Iceland", destination: "Reykjavik", startDate: .now, endDate: .now, budget: 3000)
    context.insert(trip)
    try? context.save()  // explicit save, don't rely on autosave
}
```

### Background Context with @ModelActor

For heavy work off the main thread, use `@ModelActor`.

```swift
@ModelActor
actor TripDataHandler {
    func importTrips(from data: [TripDTO]) throws {
        for dto in data {
            let trip = Trip(name: dto.name, destination: dto.destination,
                            startDate: dto.start, endDate: dto.end, budget: dto.budget)
            modelContext.insert(trip)
        }
        try modelContext.save()
    }
}

// Usage
let handler = TripDataHandler(modelContainer: container)
try await handler.importTrips(from: dtos)
```

### Context Threading Rules

**ModelContext and model instances must NEVER cross actor boundaries.** Pass `PersistentIdentifier` instead.

```swift
// CORRECT: pass ID across boundaries
let tripID = trip.persistentModelID
let handler = TripDataHandler(modelContainer: container)
await handler.updateTrip(id: tripID)

@ModelActor
actor TripDataHandler {
    func updateTrip(id: PersistentIdentifier) throws {
        guard let trip = modelContext.model(for: id) as? Trip else { return }
        trip.name = "Updated"
        try modelContext.save()
    }
}

// WRONG: passing model instance across actor boundaries
await handler.updateTrip(trip: trip)  // data race
```

**Persistent IDs are temporary before first save** — they start with a lowercase "t" prefix. Only use them as stable identifiers after `context.save()` completes.

---

## Relationships

Declare relationships with `@Relationship`. Always specify **delete rules** and **inverse** explicitly.

### One-to-Many

```swift
@Model
final class Trip {
    var name: String

    @Relationship(deleteRule: .cascade, inverse: \Activity.trip)
    var activities: [Activity] = []

    init(name: String) {
        self.name = name
    }
}

@Model
final class Activity {
    var name: String
    var trip: Trip?

    init(name: String, trip: Trip? = nil) {
        self.name = name
        self.trip = trip
    }
}
```

### One-to-One

```swift
@Model
final class User {
    var name: String

    @Relationship(deleteRule: .cascade, inverse: \Profile.user)
    var profile: Profile?

    init(name: String) {
        self.name = name
    }
}

@Model
final class Profile {
    var bio: String
    var user: User?

    init(bio: String) {
        self.bio = bio
    }
}
```

### Many-to-Many

```swift
@Model
final class Student {
    var name: String

    @Relationship(inverse: \Course.students)
    var courses: [Course] = []

    init(name: String) {
        self.name = name
    }
}

@Model
final class Course {
    var title: String
    var students: [Student] = []

    init(title: String) {
        self.title = title
    }
}
```

### Delete Rules

| Rule | Behavior |
|---|---|
| `.cascade` | Delete related objects when the owner is deleted |
| `.nullify` | Set the relationship to nil (default) — **can crash if the inverse is non-optional** |
| `.deny` | Prevent deletion if related objects exist |
| `.noAction` | Do nothing (can leave orphans) |

**Always specify delete rules explicitly.** The default `.nullify` can leave orphans or crash on non-optional relationships.

### Relationship Rules

- Put `@Relationship` on **one side only**. Putting it on both sides causes a circular reference.
- **Always specify `inverse:`** — SwiftData's auto-inference of inverse relationships is unreliable and a top source of bugs.
- Only insert the **root object** — SwiftData traverses relationships and inserts related objects automatically.
- Initialize collection relationships with empty arrays (`= []`), not optional arrays.

---

## Schema Migrations

### Lightweight Migrations

SwiftData handles these automatically with **no code required**:
- Adding a new property with a default value
- Removing a property
- Renaming a property (use `@Attribute(originalName: "oldName")`)
- Adding or removing an `@Attribute` option

### Custom Migrations with SchemaMigrationPlan

For breaking changes that require data transformation.

```swift
enum TripSchemaMigrationPlan: SchemaMigrationPlan {
    static var schemas: [any VersionedSchema.Type] {
        [SchemaV1.self, SchemaV2.self]
    }

    static var stages: [MigrationStage] {
        [migrateV1toV2]
    }

    static let migrateV1toV2 = MigrationStage.custom(
        fromVersion: SchemaV1.self,
        toVersion: SchemaV2.self
    ) { context in
        // Transform data
        let trips = try context.fetch(FetchDescriptor<SchemaV2.Trip>())
        for trip in trips {
            trip.displayName = trip.name.capitalized
        }
        try context.save()
    }
}

// Versioned schemas
enum SchemaV1: VersionedSchema {
    static var versionIdentifier = Schema.Version(1, 0, 0)
    static var models: [any PersistentModel.Type] { [Trip.self] }

    @Model
    final class Trip {
        var name: String
        init(name: String) { self.name = name }
    }
}

enum SchemaV2: VersionedSchema {
    static var versionIdentifier = Schema.Version(2, 0, 0)
    static var models: [any PersistentModel.Type] { [Trip.self] }

    @Model
    final class Trip {
        var name: String
        var displayName: String = ""
        init(name: String) {
            self.name = name
            self.displayName = name.capitalized
        }
    }
}

// Wire up the migration plan
@main
struct TripApp: App {
    var body: some Scene {
        WindowGroup { ContentView() }
            .modelContainer(for: SchemaV2.Trip.self,
                            migrationPlan: TripSchemaMigrationPlan.self)
    }
}
```

---

## Indexes for Query Performance (iOS 18+)

```swift
@Model
final class Trip {
    #Index<Trip>([\.name], [\.startDate], [\.destination, \.startDate])

    var name: String
    var destination: String
    var startDate: Date
    // ...
}
```

- Each array is a separate index. Compound indexes (multiple key paths in one array) optimize queries that filter/sort on those properties together.
- Indexes have a small **write-time cost**. Do not index write-heavy / read-rare data (e.g., logs).
- Index the properties you use in `#Predicate` and `SortDescriptor`.

---

## iCloud Sync with SwiftData

SwiftData uses CloudKit under the hood. Enable by adding the iCloud capability with CloudKit and Background Modes (remote notifications).

```swift
// Default container — syncs automatically if iCloud capability is enabled
.modelContainer(for: Trip.self)

// Explicit control over sync
let config = ModelConfiguration(
    cloudKitDatabase: .private("iCloud.com.yourapp.trips")  // or .none to disable
)
```

### CloudKit Compatibility Constraints

These constraints are **hard requirements** — violating them causes silent sync failures or crashes:

1. **No `@Attribute(.unique)` or `#Unique`** — CloudKit does not support uniqueness constraints
2. **All properties must have default values or be optional** — CloudKit records can arrive partially
3. **All relationships must be optional** — related records may sync out of order
4. **No required (non-optional) relationship inverses** — use optional on both sides
5. CloudKit is **eventually consistent** — code must handle missing or stale data gracefully
6. **Production CloudKit schemas are additive-only** after promotion — you cannot remove or rename fields

```swift
// CloudKit-compatible model
@Model
final class Trip {
    var name: String = ""               // default value
    var destination: String = ""         // default value
    var startDate: Date = Date.now       // default value
    var notes: String?                   // optional

    @Relationship(deleteRule: .nullify)
    var activities: [Activity]? = []     // optional relationship

    init(name: String, destination: String, startDate: Date) {
        self.name = name
        self.destination = destination
        self.startDate = startDate
    }
}
```

---

## Fetching Without @Query

For ViewModels, services, or background work, use `FetchDescriptor` directly.

```swift
@MainActor
@Observable
final class TripViewModel {
    var trips: [Trip] = []
    private let context: ModelContext

    init(context: ModelContext) {
        self.context = context
    }

    func loadUpcomingTrips() throws {
        var descriptor = FetchDescriptor<Trip>(
            predicate: #Predicate { $0.startDate > Date.now },
            sortBy: [SortDescriptor(\.startDate)]
        )
        descriptor.fetchLimit = 20
        descriptor.relationshipKeyPathsForPrefetching = [\.activities]
        trips = try context.fetch(descriptor)
    }
}
```

### Performance Controls

| Property | Purpose |
|---|---|
| `fetchLimit` | Limit number of results |
| `fetchOffset` | Skip N results (pagination) |
| `propertiesToFetch` | Partial fetch — only load specified properties |
| `relationshipKeyPathsForPrefetching` | Prefetch relationships to avoid N+1 queries |
| `includePendingChanges` | Include unsaved changes in results (default: `true`) |

**Note:** `fetchCount()` returns a count without loading objects, but it does **not** live-update like `@Query`.

---

## Batch Deletion

```swift
// Safe batch delete with predicate
try context.delete(model: Trip.self, where: #Predicate<Trip> { $0.startDate < cutoffDate })

// DANGER: no predicate = deletes ALL instances of the model
try context.delete(model: Trip.self)  // deletes every Trip
```

---

## Common LLM Mistakes

### 1. Using `NSPredicate` Instead of `#Predicate`
SwiftData uses the `#Predicate` macro. `NSPredicate` is Core Data.

### 2. Property Named `description`
`description` conflicts with `CustomStringConvertible`. **Never use `description` as a property name on `@Model` classes.**

### 3. Property Observers on @Model
`willSet` and `didSet` are **silently ignored** on `@Model` properties. The macro replaces stored property access. Use explicit methods instead.

```swift
// WRONG: observers are silently ignored
@Model
final class Trip {
    var budget: Decimal {
        didSet { lastModified = .now }  // never fires
    }
    var lastModified: Date = .now
}

// CORRECT: use a method
@Model
final class Trip {
    var budget: Decimal
    var lastModified: Date = .now

    func updateBudget(_ newBudget: Decimal) {
        budget = newBudget
        lastModified = .now
    }
}
```

### 4. @Relationship on Both Sides
Put `@Relationship` on **one** side. Declaring it on both creates a circular reference.

### 5. Missing Inverse Specification
Always use `inverse:` on `@Relationship`. SwiftData's auto-inference is buggy and a top source of data corruption.

### 6. Passing Model Objects Across Actor Boundaries
Pass `persistentModelID`, not the model instance. Model objects are not thread-safe.

### 7. Using @Query Outside SwiftUI Views
`@Query` only works in `View` conformances. Use `FetchDescriptor` in ViewModels and services.

### 8. Relying on Autosave for Critical Writes
Autosave timing is unpredictable. Always call `try context.save()` explicitly after important mutations.

### 9. Forgetting Default Values with CloudKit
All properties need defaults or optionality for CloudKit sync. Missing defaults cause silent sync failures.

### 10. No Need to Check `hasChanges` Before Saving
Calling `save()` on a clean context is a no-op. Don't guard it.

---

## Key Rules

- [ ] Use `@Model` on `final class` — SwiftData models are reference types, not value types
- [ ] Always specify `inverse:` on `@Relationship` — never rely on auto-inference
- [ ] Always specify explicit delete rules on relationships
- [ ] Use `#Predicate`, never `NSPredicate`
- [ ] Call `try context.save()` explicitly — don't rely on autosave
- [ ] Pass `persistentModelID` across actor boundaries, not model instances
- [ ] Use `@ModelActor` for background persistence work
- [ ] Use `@Attribute(originalName:)` for property renames instead of migrations
- [ ] Initialize collection relationships with `= []`, not as optionals (unless CloudKit)
- [ ] Test predicates at runtime — compilation does not guarantee they work

## Mistakes to Avoid

- [ ] Naming a property `description` on an `@Model` class
- [ ] Adding property observers (`willSet`/`didSet`) to `@Model` properties
- [ ] Putting `@Relationship` on both sides of a relationship
- [ ] Using `@Attribute(.unique)` or `#Unique` with CloudKit sync
- [ ] Using `hasPrefix()`/`hasSuffix()` in `#Predicate` (use `starts(with:)`)
- [ ] Using `.isEmpty == false` in `#Predicate` (use `!.isEmpty`)
- [ ] Using computed properties or `@Transient` properties in `#Predicate`
- [ ] Using `@Query` in a ViewModel or service — it only works in SwiftUI views
- [ ] Passing model instances across actor boundaries instead of `PersistentIdentifier`
- [ ] Multiple `#Unique` macros on one model — only the last one applies
