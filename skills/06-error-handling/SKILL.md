---
description: Apply Swift error handling best practices
user_invocable: true
---

# Skill 06 — Error Handling Best Practices

**Source:** Swift.org, Apple Documentation, Community  
**Applies to:** Service layer, networking, data parsing

---

## Domain-Specific Error Types

```swift
enum ServiceError: Error, LocalizedError {
    case invalidInput
    case networkFailure(underlying: Error)
    case invalidResponse
    case decodingFailed(underlying: Error)
    case rateLimited

    var errorDescription: String? {
        switch self {
        case .invalidInput: "Please check your input and try again."
        case .networkFailure: "Unable to connect. Check your internet."
        case .invalidResponse: "Received an unexpected response."
        case .decodingFailed: "Could not parse the server response."
        case .rateLimited: "Too many requests. Please wait a moment."
        }
    }
}
```

## Typed Throws (Swift 6+)

```swift
func validate(input: String) throws(ServiceError) {
    guard !input.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty else {
        throw .invalidInput
    }
}

// Caller gets exhaustive switch
do throws(ServiceError) {
    try validate(input: text)
} catch {
    switch error {
    case .invalidInput: showValidationAlert()
    case .networkFailure: showRetryOption()
    // ... compiler ensures all cases handled
    }
}
```

## Separate Log vs User Messages

```swift
protocol AppError: Error {
    var logMessage: String { get }         // For developers/logs
    var userFriendlyMessage: String { get } // For UI display
}
```

## Error Handling Patterns

```swift
// Use guard + try for early exits
func fetchItem(by id: String) async throws -> Item {
    guard !id.isEmpty else { throw ServiceError.invalidInput }

    let data = try await fetchFromAPI(id)
    let result = try decode(data)
    return result
}

// Propagate errors — don't catch everything at the call site
// Let the ViewModel handle presentation logic
```

## Key Rules

- [ ] Create domain-specific error types — don't reuse generic errors
- [ ] Conform to `LocalizedError` for user-facing messages
- [ ] Use `guard` with `try` for early exits
- [ ] Use `try?` only for genuinely non-critical failures
- [ ] Never use `try!` in production code (acceptable in tests/previews)
- [ ] Propagate errors at the right level — let ViewModels handle presentation

## Mistakes to Avoid

- [ ] Generic `catch` that swallows error details
- [ ] Using `try?` to silence errors that should be handled
- [ ] `try!` in production code
- [ ] Throwing `NSError` or raw `Error` instead of domain types
- [ ] Mixing log messages with user-facing messages
