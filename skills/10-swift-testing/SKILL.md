---
description: Write tests using the Swift Testing framework
user_invocable: true
---

# Skill 10 — Swift Testing Framework

**Source:** Apple WWDC 2024-2025, Swift.org  
**Applies to:** Unit tests, integration tests

---

## Swift Testing Is the Default (Xcode 16+)

Use Swift Testing for all new tests. Keep XCTest only for UI tests (`XCUITest`) and performance tests.

## Basic Structure

```swift
import Testing

@Suite("Item Service Tests")
struct ItemServiceTests {

    @Test("Fetching by ID returns correct item")
    func fetchByID() async throws {
        let service = MockDataService()
        let result = try await service.fetchItem(by: "123")
        #expect(result.title == "Expected Title")
    }

    @Test("Empty input throws error")
    func emptyInput() {
        #expect(throws: ServiceError.invalidInput) {
            try validate(input: "")
        }
    }
}
```

## #expect vs #require

```swift
@Test func resultHasValidData() throws {
    let item = try #require(viewModel.selectedItem) // stops test if nil
    #expect(item.status == .active)          // reports failure, continues
    #expect(item.priority > 0)               // still runs even if above fails
}
```

## Parameterized Tests

```swift
@Test("Status mapping", arguments: [
    ("pending", Status.pending),
    ("active", Status.active),
    ("completed", Status.completed),
    ("archived", Status.archived),
])
func statusMapping(input: String, expected: Status) async throws {
    let result = try await service.parseStatus(from: input)
    #expect(result == expected)
}
```

## Traits

```swift
@Test(.timeLimit(.minutes(1)))
func apiCallCompletes() async throws { ... }

@Test(.disabled("API key not available in CI"))
func integrationTest() async throws { ... }

@Test(.bug("https://github.com/org/repo/issues/42"))
func regressionFix() { ... }
```

## Tags for Organization

```swift
extension Tag {
    @Tag static var services: Self
    @Tag static var viewModels: Self
    @Tag static var integration: Self
}

@Suite(.tags(.services))
struct MockServiceTests { ... }
```

## Setup via Init

```swift
@Suite struct ViewModelTests {
    let viewModel: ItemListViewModel

    init() {
        viewModel = ItemListViewModel(service: MockDataService())
    }

    @Test func initialStateIsEmpty() {
        #expect(viewModel.selectedItem == nil)
        #expect(viewModel.isLoading == false)
    }
}
```

## Key Rules

- [ ] Use Swift Testing for all new unit tests
- [ ] Use `#require` for preconditions, `#expect` for assertions
- [ ] Use parameterized tests to cover multiple inputs
- [ ] Tag tests for organization (`@Tag`)
- [ ] Tests run in parallel by default — avoid shared mutable state

## Mistakes to Avoid

- [ ] Mixing XCTest assertions (`XCTAssertEqual`) inside Swift Testing tests
- [ ] Not using `#require` for preconditions, leading to cascading nil failures
- [ ] Shared mutable state causing flaky parallel tests
- [ ] Forgetting `.serialized` trait when tests must run sequentially
