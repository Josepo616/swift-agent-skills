---
description: MVVM + Service-Oriented Architecture for SwiftUI apps
user_invocable: true
---

# Skill 15 — MVVM + Service Architecture

**Source:** Ravn iOS Team Practices, Community Patterns
**Applies to:** App architecture, feature design, testability

---

## Overview

This skill defines an opinionated architecture: **MVVM with a Service-Oriented model layer**. It is not a survey of patterns. Use this as the default architecture for SwiftUI apps unless the project explicitly requires something else (TCA, Clean Architecture, etc.).

The core idea: **MVVM alone has a weak model layer.** Services fill that gap — they encapsulate business logic, data access, and side effects behind protocol contracts. ViewModels orchestrate services; Views render state.

---

## Layer Responsibilities

```
┌─────────────────────────────────────────────┐
│  View                                       │
│  Renders UI. Owns zero business logic.      │
│  Reads ViewModel state. Sends user actions. │
├─────────────────────────────────────────────┤
│  ViewModel                                  │
│  Owns screen state. Calls services.         │
│  Transforms data for display.               │
│  One ViewModel per View. Always @MainActor. │
├─────────────────────────────────────────────┤
│  Service (Protocol + Implementation)        │
│  Encapsulates a business capability.        │
│  Networking, persistence, auth, analytics.  │
│  Stateless or owns internal state only.     │
├─────────────────────────────────────────────┤
│  Model                                      │
│  Plain data types. Structs + enums.         │
│  Codable, Sendable, Identifiable.           │
└─────────────────────────────────────────────┘
```

**Dependency direction:** View → ViewModel → Service → Model. Never reverse this flow.

---

## The 1 View : 1 ViewModel Rule

Every screen gets its own dedicated ViewModel. This is strict — no sharing.

```swift
// ✅ Correct: each screen owns its ViewModel
struct OrderListView: View {
    @State private var viewModel = OrderListViewModel()
    var body: some View { /* ... */ }
}

struct OrderDetailView: View {
    @State private var viewModel: OrderDetailViewModel

    init(order: Order) {
        self._viewModel = State(initialValue: OrderDetailViewModel(order: order))
    }

    var body: some View { /* ... */ }
}
```

```swift
// ❌ Wrong: sharing a ViewModel between two screens
struct OrderListView: View {
    @State private var viewModel = OrderViewModel() // shared "god" ViewModel

    var body: some View {
        NavigationLink {
            OrderDetailView(viewModel: viewModel) // passing the same VM
        }
    }
}
```

**Why shared ViewModels are an anti-pattern:**
- They grow into "God objects" that own state for multiple screens
- Changing one screen's logic risks breaking another
- They make navigation harder — the ViewModel outlives the screen it was meant for
- Testing becomes tangled — you test two screens' logic in one object

**If two screens need the same data**, they share a Service, not a ViewModel. Each ViewModel calls the service independently.

---

## Service Design

A service encapsulates one business capability. It is defined by a protocol contract and has a concrete implementation.

### Protocol Contract

```swift
protocol OrderServiceProtocol: Sendable {
    func fetchOrders() async throws -> [Order]
    func fetchOrder(id: Order.ID) async throws -> Order
    func cancelOrder(id: Order.ID) async throws
}
```

### Implementation

```swift
final class OrderService: OrderServiceProtocol {
    private let apiClient: APIClientProtocol

    init(apiClient: APIClientProtocol) {
        self.apiClient = apiClient
    }

    func fetchOrders() async throws -> [Order] {
        try await apiClient.request(OrderEndpoints.list)
    }

    func fetchOrder(id: Order.ID) async throws -> Order {
        try await apiClient.request(OrderEndpoints.detail(id))
    }

    func cancelOrder(id: Order.ID) async throws {
        try await apiClient.request(OrderEndpoints.cancel(id))
    }
}
```

### Service Rules

- **Protocol first.** Every service has a protocol. This enables testing and substitution.
- **Self-contained.** A service owns its logic. It does not reach into other services' internals.
- **Sendable.** Services cross concurrency boundaries. Mark them `Sendable` or use actors.
- **Composable.** Services can depend on other services — always via their protocol.
- **Stateless by default.** If a service must hold state (e.g., a cache), make it an `actor`.

---

## Service Composition

Services can depend on other services through their protocol contracts.

```swift
protocol PaymentServiceProtocol: Sendable {
    func processPayment(for order: Order) async throws -> PaymentReceipt
}

final class PaymentService: PaymentServiceProtocol {
    private let orderService: OrderServiceProtocol
    private let analyticsService: AnalyticsServiceProtocol

    init(
        orderService: OrderServiceProtocol,
        analyticsService: AnalyticsServiceProtocol
    ) {
        self.orderService = orderService
        self.analyticsService = analyticsService
    }

    func processPayment(for order: Order) async throws -> PaymentReceipt {
        let receipt = try await chargePayment(order)
        analyticsService.track(.paymentCompleted(orderId: order.id))
        return receipt
    }

    private func chargePayment(_ order: Order) async throws -> PaymentReceipt {
        // Payment processing logic
    }
}
```

**Never create circular dependencies.** If Service A needs Service B and B needs A, extract the shared logic into a third service.

---

## Services as Swift Packages

For large projects, extract services into Swift packages to enforce module boundaries.

```
MyApp/
├── App/                           # App target
│   ├── MyApp.swift
│   └── Features/
├── Packages/
│   ├── OrderService/              # Swift package
│   │   ├── Sources/
│   │   │   ├── OrderServiceProtocol.swift
│   │   │   └── OrderService.swift
│   │   └── Tests/
│   ├── PaymentService/            # Swift package
│   │   ├── Sources/
│   │   └── Tests/
│   └── Core/                      # Shared models, API client
│       ├── Sources/
│       └── Tests/
```

Benefits:
- Compile-time enforcement of dependency direction
- Parallel builds (faster CI)
- Services are reusable across targets (app, widget, extension)
- Each package has its own test target

---

## Feature Folder Structure

Organize each feature as a self-contained folder with four layers:

```
Features/
└── Orders/
    ├── Model/
    │   ├── Order.swift
    │   └── OrderStatus.swift
    ├── Service/
    │   ├── OrderServiceProtocol.swift
    │   └── OrderService.swift
    ├── ViewModel/
    │   ├── OrderListViewModel.swift
    │   └── OrderDetailViewModel.swift
    └── View/
        ├── OrderListView.swift
        ├── OrderDetailView.swift
        └── Components/
            ├── OrderCard.swift
            └── OrderStatusBadge.swift
```

**Rules:**
- Each feature folder maps to one user-facing flow
- Models are plain structs — no framework imports
- Services own the "how" (API calls, persistence, caching)
- ViewModels own the "what" (screen state, loading, errors)
- Views own the "look" (layout, styling, animations)
- Shared UI components that appear in multiple features go in a top-level `Components/` folder

---

## ViewModel Pattern

### Structure

```swift
import SwiftUI

@Observable
@MainActor
final class OrderListViewModel {
    // MARK: - State

    private(set) var orders: [Order] = []
    private(set) var isLoading = false
    private(set) var error: AppError?

    var hasError: Bool { error != nil }

    // MARK: - Dependencies

    private let orderService: OrderServiceProtocol

    // MARK: - Tasks

    private var fetchTask: Task<Void, Never>?

    // MARK: - Init

    init(orderService: OrderServiceProtocol = OrderService()) {
        self.orderService = orderService
    }

    // MARK: - Actions

    func onAppear() {
        fetchOrders()
    }

    func refresh() {
        fetchOrders()
    }

    func deleteOrder(_ order: Order) async {
        do {
            try await orderService.cancelOrder(id: order.id)
            orders.removeAll { $0.id == order.id }
        } catch {
            self.error = AppError(error)
        }
    }

    // MARK: - Private

    private func fetchOrders() {
        fetchTask?.cancel()
        fetchTask = Task {
            isLoading = true
            defer { isLoading = false }

            do {
                orders = try await orderService.fetchOrders()
            } catch is CancellationError {
                // Task was cancelled — do nothing
            } catch {
                self.error = AppError(error)
            }
        }
    }

    deinit {
        fetchTask?.cancel()
    }
}
```

### ViewModel Rules

- **Always `@Observable` and `@MainActor`.** The ViewModel is a UI-layer object.
- **`private(set)` for state.** Views read state; only the ViewModel mutates it.
- **Cancel stale work.** When starting a new fetch, cancel the previous task.
- **Clean up in `deinit`.** Cancel any running tasks.
- **Default parameter for production dependency.** Tests override via `init`.
- **No `import SwiftUI` view types.** ViewModels must not reference `Color`, `Image`, `Font`, or any view types. They expose data; the View decides presentation.

---

## ViewModel-to-Service Communication

### Pattern 1: Direct Async Calls (Default)

The ViewModel calls service methods and updates its own state.

```swift
func loadProfile() async {
    isLoading = true
    defer { isLoading = false }

    do {
        profile = try await profileService.fetchProfile()
    } catch {
        self.error = AppError(error)
    }
}
```

### Pattern 2: Streaming with AsyncSequence

For real-time data (WebSocket, database observation), the service exposes an `AsyncSequence`.

```swift
// Service
protocol ChatServiceProtocol: Sendable {
    var messages: AsyncStream<[Message]> { get }
    func send(_ text: String) async throws
}

// ViewModel
func observeMessages() {
    observeTask?.cancel()
    observeTask = Task {
        for await newMessages in chatService.messages {
            self.messages = newMessages
        }
    }
}
```

### Pattern 3: Callback for One-Shot Events

For navigation triggers or alerts after an action completes, use a closure or published property.

```swift
@Observable
@MainActor
final class CheckoutViewModel {
    var navigateToConfirmation: Order?

    func placeOrder() async {
        do {
            let order = try await orderService.placeOrder(cart: cart)
            navigateToConfirmation = order
        } catch {
            self.error = AppError(error)
        }
    }
}
```

---

## View Integration

```swift
struct OrderListView: View {
    @State private var viewModel = OrderListViewModel()

    var body: some View {
        List {
            ForEach(viewModel.orders) { order in
                NavigationLink(value: order) {
                    OrderCard(order: order)
                }
            }
        }
        .overlay {
            if viewModel.isLoading && viewModel.orders.isEmpty {
                ProgressView()
            }
        }
        .refreshable {
            viewModel.refresh()
        }
        .task {
            viewModel.onAppear()
        }
        .alert(
            "Error",
            isPresented: $viewModel.hasError,
            presenting: viewModel.error
        ) { _ in
            Button("OK") { viewModel.error = nil }
        } message: { error in
            Text(error.localizedDescription)
        }
    }
}
```

**View rules:**
- Use `@State` to own the ViewModel (iOS 17+ with `@Observable`)
- Call ViewModel methods for actions — never put business logic in the View
- Use `.task` for initial data loading — it auto-cancels when the view disappears
- Keep `body` focused on layout; extract subviews for complex sections

---

## Shared Managers vs. Dedicated ViewModels

Not everything fits the 1:1 rule. Some state is truly app-wide. Use a shared manager for:

| Use Case | Pattern | Example |
|----------|---------|---------|
| Auth state | Shared `@Observable` manager via `@Environment` | `SessionManager` |
| Navigation state | Shared `@Observable` manager at root | `NavigationState` |
| Feature flags | Shared manager via `@Environment` | `FeatureFlagManager` |
| Screen-specific logic | Dedicated ViewModel | `OrderListViewModel` |

```swift
// Shared manager — injected via environment
@Observable
final class SessionManager {
    private(set) var currentUser: User?
    var isAuthenticated: Bool { currentUser != nil }

    private let authService: AuthServiceProtocol

    init(authService: AuthServiceProtocol = AuthService()) {
        self.authService = authService
    }

    func signIn(email: String, password: String) async throws {
        currentUser = try await authService.signIn(email: email, password: password)
    }

    func signOut() {
        currentUser = nil
    }
}

// Inject at app root
@main
struct MyApp: App {
    @State private var sessionManager = SessionManager()

    var body: some Scene {
        WindowGroup {
            RootView()
                .environment(sessionManager)
        }
    }
}

// ViewModel reads it via init, NOT via @Environment
@Observable
@MainActor
final class ProfileViewModel {
    private let sessionManager: SessionManager

    init(sessionManager: SessionManager) {
        self.sessionManager = sessionManager
    }
}

// View bridges the gap
struct ProfileView: View {
    @Environment(SessionManager.self) private var sessionManager
    @State private var viewModel: ProfileViewModel?

    var body: some View {
        Group {
            if let viewModel {
                ProfileContent(viewModel: viewModel)
            }
        }
        .task {
            viewModel = ProfileViewModel(sessionManager: sessionManager)
        }
    }
}
```

**Rule:** Shared managers are injected via `@Environment`. ViewModels receive them through `init` — ViewModels must never use `@Environment` directly.

---

## Testing: Mock Services via Protocol Conformance

Every service has a protocol. Every test creates a mock that conforms to it.

### Mock Service

```swift
final class MockOrderService: OrderServiceProtocol {
    var ordersToReturn: [Order] = []
    var errorToThrow: Error?
    var cancelOrderCallCount = 0

    func fetchOrders() async throws -> [Order] {
        if let error = errorToThrow { throw error }
        return ordersToReturn
    }

    func fetchOrder(id: Order.ID) async throws -> Order {
        if let error = errorToThrow { throw error }
        guard let order = ordersToReturn.first(where: { $0.id == id }) else {
            throw AppError.notFound
        }
        return order
    }

    func cancelOrder(id: Order.ID) async throws {
        cancelOrderCallCount += 1
        if let error = errorToThrow { throw error }
    }
}
```

### ViewModel Test

```swift
import Testing

@Suite("OrderListViewModel")
struct OrderListViewModelTests {
    let mockService = MockOrderService()

    @Test
    @MainActor
    func fetchOrders_success_updatesState() async {
        let expected = [Order.mock(), Order.mock()]
        mockService.ordersToReturn = expected

        let viewModel = OrderListViewModel(orderService: mockService)
        viewModel.onAppear()

        // Wait for the internal task to complete
        try? await Task.sleep(for: .milliseconds(50))

        #expect(viewModel.orders == expected)
        #expect(viewModel.isLoading == false)
        #expect(viewModel.error == nil)
    }

    @Test
    @MainActor
    func fetchOrders_failure_setsError() async {
        mockService.errorToThrow = AppError.networkError

        let viewModel = OrderListViewModel(orderService: mockService)
        viewModel.onAppear()

        try? await Task.sleep(for: .milliseconds(50))

        #expect(viewModel.orders.isEmpty)
        #expect(viewModel.error != nil)
    }

    @Test
    @MainActor
    func deleteOrder_callsService() async {
        let order = Order.mock()
        mockService.ordersToReturn = [order]

        let viewModel = OrderListViewModel(orderService: mockService)
        viewModel.onAppear()
        try? await Task.sleep(for: .milliseconds(50))

        await viewModel.deleteOrder(order)

        #expect(mockService.cancelOrderCallCount == 1)
        #expect(viewModel.orders.isEmpty)
    }
}
```

---

## Loadable State Pattern

Use an enum to model async data states explicitly. This prevents impossible states (loading + error simultaneously).

```swift
enum Loadable<Value> {
    case idle
    case loading
    case loaded(Value)
    case failed(AppError)

    var value: Value? {
        if case .loaded(let value) = self { return value }
        return nil
    }

    var isLoading: Bool {
        if case .loading = self { return true }
        return false
    }

    var error: AppError? {
        if case .failed(let error) = self { return error }
        return nil
    }
}

// Usage in ViewModel
@Observable
@MainActor
final class OrderListViewModel {
    private(set) var state: Loadable<[Order]> = .idle
    private let orderService: OrderServiceProtocol

    init(orderService: OrderServiceProtocol = OrderService()) {
        self.orderService = orderService
    }

    func onAppear() {
        guard case .idle = state else { return }
        fetchOrders()
    }

    private func fetchOrders() {
        state = .loading
        Task {
            do {
                let orders = try await orderService.fetchOrders()
                state = .loaded(orders)
            } catch {
                state = .failed(AppError(error))
            }
        }
    }
}
```

---

## Common Mistakes

- [ ] **God ViewModel.** A ViewModel that handles logic for multiple screens. Split it — one ViewModel per screen.
- [ ] **Service with no protocol.** Concrete service types injected directly make testing impossible. Always define a protocol.
- [ ] **ViewModel using `@Environment`.** ViewModels are not views. Pass dependencies via `init`, not property wrappers.
- [ ] **Business logic in the View.** Filtering, sorting, validation, or API calls in `body` or action closures. Move to ViewModel or Service.
- [ ] **Circular service dependencies.** Service A depends on B, B depends on A. Extract shared logic into a third service.
- [ ] **Forgetting task cancellation.** Starting a new fetch without cancelling the previous one leads to race conditions and stale data overwrites.
- [ ] **Mutable ViewModel state without `private(set)`.** Views should never directly mutate ViewModel state — they call methods instead.
- [ ] **Using `ObservableObject` + `@Published` on iOS 17+.** Use `@Observable` instead — it provides more granular view updates and cleaner syntax.
- [ ] **Shared ViewModel passed between parent and child views.** Use a shared Service instead; each view gets its own ViewModel that calls the service.
- [ ] **Service that imports SwiftUI.** Services are pure Swift. They must not depend on UI frameworks.
