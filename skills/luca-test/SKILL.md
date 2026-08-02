---
name: luca-test
description: Writes unit tests for LUCA Service and Store components using Swift Testing, with proper naming conventions and dependency mocking patterns.
---

You are an expert in writing unit tests for the **LUCA architecture**. When the user provides a Store or Service implementation, generate comprehensive Swift Testing tests following all LUCA test conventions below.

Ask the user to share the Store/Service code they want to test if not already provided.

---

## Test Scope

| Target    | Location                  | Test file location               |
| --------- | ------------------------- | -------------------------------- |
| `Service` | `Sources/Model/Services/` | `Tests/ModelTests/ServiceTests/` |
| `Store`   | `Sources/Model/Stores/`   | `Tests/ModelTests/StoreTests/`   |

Do **not** write UI tests or snapshot tests — unit tests for Services and Stores are sufficient to cover app functionality in LUCA.

---

## Framework

Use **Swift Testing** (not XCTest):

```swift
import Testing
import DataSource
@testable import Model
```

Mark test structs/classes with `@MainActor` when testing Stores (since Stores are `@MainActor`):

```swift
@MainActor struct FooStoreTests {
    @Test
    func send_viewAppeared_itemsAreLoaded() async { ... }
}
```

---

## Test Naming Convention

```
func {methodOrAction}_{condition_optional}_{expectedResult}()
```

**Store tests (Action-based):**

```swift
func send_viewAppeared_savedDataIsRestored()
func send_deleteButtonTapped_itemIsSelected_itemIsDeleted()
func send_notificationsToggleSwitched_notificationsAreDisabled_notificationsAreEnabled()
func send_themePickerSelected_themeIsUpdated()
func send_fetchDataResponse_success_itemsArePopulated()
func send_fetchDataResponse_failure_errorIsShown()
```

**Service tests (method-based):**

```swift
func calculateBMI_heightIsNonZero_correctValueIsReturned()
func calculateBMI_heightIsZero_zeroIsReturned()
func processInput_emptyString_emptyStringIsReturned()
```

---

## Mocking Dependencies

### Using `testDependency(of:injection:)` (single client)

```swift
let client = testDependency(of: UserDefaultsClient.self) {
    $0.data = { _ in
        try! JSONEncoder().encode(Foo(name: "test", value: 42))
    }
}
```

### Using `AppDependencies.testDependencies(...)` (full dependency set)

```swift
let deps = AppDependencies.testDependencies(
    userDefaultsClient: testDependency(of: UserDefaultsClient.self) {
        $0.data = { _ in
            try! JSONEncoder().encode(Foo(name: "test", value: 42))
        }
    }
)
```

### Capturing side effects

Use `OSAllocatedUnfairLock` to safely capture values from `@Sendable` closures:

```swift
import os

let captured = OSAllocatedUnfairLock<Data?>(initialState: nil)
let deps = AppDependencies.testDependencies(
    userDefaultsClient: testDependency(of: UserDefaultsClient.self) {
        $0.setData = { data, _ in
            captured.withLock { $0 = data }
        }
    }
)
// After test action:
#expect(captured.withLock(\.self) != nil)
```

### Mocking `AppStateClient` with inspectable state

```swift
let appState = OSAllocatedUnfairLock<AppState>(initialState: .init(hasAlreadyBootstrap: false))
let deps = AppDependencies.testDependencies(
    appStateClient: .testDependency(appState)
)
// After test action:
#expect(appState.withLock(\.hasAlreadyBootstrap) == true)
```

### Stateful clients & repositories

A stateful, side-effecting dependency (UserDefaults, a file store, the keychain) does **not** need a
bespoke stateful fake class or a shared storage helper. Build the minimal state inline per test with
`OSAllocatedUnfairLock` + `testDependency(of:)`, choosing the shape by what you verify:

- **Verify a write** → capture the setter into a lock and assert the recorded call (no stored state).
- **Provide a read** → stub the getter to return the expected value (`$0.integer = { _ in 3 }`).
- **In-flow round-trip** (a `reduce`/Service persists a value then immediately re-reads it) → back that
  **one key** with a lock so `set` → `get` connect:

```swift
let stored = OSAllocatedUnfairLock<Int>(initialState: 0)
let client = testDependency(of: UserDefaultsClient.self) {
    $0.integer = { key in key == .triggerMethod ? stored.withLock(\.self) : 0 }
    $0.set = { value, key in
        let raw = value as? Int            // cast OUTSIDE withLock — `value: Any?` is not Sendable
        if key == .triggerMethod { stored.withLock { $0 = raw } }
    }
}
```

A **Repository is composed of clients, not a test seam**: construct the real Repository over a mocked
Client and observe the client interaction — don't read persistence back through a *second* Repository
instance. For **migration / transform** logic (read legacy → write current → delete legacy), stub the
legacy reads and capture the `set`/`removeObject` calls; no key-value store is required.

---

## Service Test Pattern

Services are stateless structs — test them as pure functions:

```swift
import Testing
import DataSource
@testable import Model

@MainActor struct BMIServiceTests {
    @Test
    func calculateBMI_heightIsNonZero_correctValueIsReturned() {
        let sut = BMIService(.testDependencies())
        let result = sut.calculateBMI(weight: 70, height: 175)
        #expect(result == 22.86)
    }

    @Test
    func calculateBMI_heightIsZero_zeroIsReturned() {
        let sut = BMIService(.testDependencies())
        let result = sut.calculateBMI(weight: 70, height: 0)
        #expect(result == 0)
    }
}
```

---

## TestStore

`TestStore` is a testing utility generated in `Tests/ModelTests/TestStore.swift`. It wraps a `Composable` store, intercepts the parent `action` callback, and records all delegated actions in `actionHistory`.

**When to use:**

- Basic state mutation / side-effect tests → instantiate the Store directly (no TestStore needed)
- Tests that must verify **which actions the Store delegates to its parent** → use TestStore

**Initialization — always use the factory pattern:**

```swift
let sut = TestStore { action in
    FooStore(.testDependencies(), action: action)
}
```

The `action` closure passed to the factory is TestStore's interceptor. Any action the Store forwards to its parent via `Composable.send` is recorded.

**Verifying delegated actions with `receive`:**

```swift
await sut.send(.closeButtonTapped)
sut.receive {
    if case .closeButtonTapped = $0 { true } else { false }
}
```

- `send` dispatches the action and then removes the directly passed-through action (since `Composable.send` always calls `action` with the same value)
- `receive` verifies that a specific action appeared in `actionHistory` from an async internal dispatch; call it once per expected delegation before the next `send`
- `Action` is not required to be `Equatable` — always use pattern matching (`if case`) inside the closure
- If `actionHistory` is non-empty when `send` is called again, the test fails automatically

**Accessing Store properties via dynamic member lookup:**

```swift
#expect(sut.items.count == 3)   // proxies wrappedStore.items
```

---

## Store Test Patterns

### Basic read test

```swift
@MainActor struct FooStoreTests {
    @Test
    func send_viewAppeared_savedDataIsRestored() async {
        let expected = Foo(name: "Alice", value: 42)
        let sut = FooStore(
            .testDependencies(
                userDefaultsClient: testDependency(of: UserDefaultsClient.self) {
                    $0.data = { _ in try! JSONEncoder().encode(expected) }
                }
            ),
            action: { _ in }
        )
        await sut.send(.viewAppeared)
        #expect(sut.foo == expected)
    }
}
```

### State mutation test

```swift
@Test
func send_calculateButtonTapped_bmiIsCalculated() async {
    let sut = FooStore(
        .testDependencies(),
        person: Person(name: "Bob", weight: 70.0, height: 175.0),
        action: { _ in }
    )
    await sut.send(.calculateButtonTapped)
    #expect(sut.calculatedBMI == 22.86)
}
```

### Side-effect capture test

```swift
@Test
func send_saveButtonTapped_dataIsPersisted() async {
    let captured = OSAllocatedUnfairLock<Data?>(initialState: nil)
    let sut = FooStore(
        .testDependencies(
            userDefaultsClient: testDependency(of: UserDefaultsClient.self) {
                $0.setData = { data, _ in captured.withLock { $0 = data } }
            }
        ),
        foo: Foo(name: "Alice", value: 42),
        action: { _ in }
    )
    await sut.send(.saveButtonTapped)
    #expect(captured.withLock(\.self) != nil)
}
```

### AppState mutation test

```swift
@Test
func send_viewAppeared_tutorialIsShownOnFirstLaunch() async {
    let appState = OSAllocatedUnfairLock<AppState>(initialState: .init(hasAlreadyTutorial: false))
    let sut = FooStore(
        .testDependencies(appStateClient: .testDependency(appState)),
        action: { _ in }
    )
    await sut.send(.viewAppeared)
    #expect(sut.isPresentedTutorial == true)
    #expect(appState.withLock(\.hasAlreadyTutorial) == true)
}
```

### Stream producer test (Service → `AsyncStreamBundle`)

A Service that pushes into an `AppState` stream is verified **synchronously** via `latestValue` — no waiting needed:

```swift
@Test
func increment_sends_incremented_count() {
    let appState = OSAllocatedUnfairLock<AppState>(initialState: .init())
    let sut = CounterService(.testDependencies(appStateClient: .testDependency(appState)))
    sut.increment()
    sut.increment()
    #expect(appState.withLock(\.count.latestValue) == 2)
}
```

### Stream consumer test (Store subscription)

A Store that subscribes to a stream reacts **asynchronously**, so push a value then poll with `waitUntil`. Always send `.viewDisappeared` at the end to cancel the subscription.

Add this helper once at `Tests/ModelTests/WaitUntil.swift`:

```swift
import Foundation

@discardableResult
func waitUntil(_ condition: @MainActor () -> Bool) async -> Bool {
    for _ in 0 ..< 200 {
        if await condition() { return true }
        try? await Task.sleep(for: .milliseconds(10))
    }
    return false
}
```

```swift
@MainActor @Test
func observes_count_stream() async {
    let appState = OSAllocatedUnfairLock<AppState>(initialState: .init())
    let sut = ContentStore(.testDependencies(appStateClient: .testDependency(appState)))
    await sut.send(.viewAppeared("Test"))
    appState.withLock { $0.count.send(5) }
    await waitUntil { sut.count == 5 }
    #expect(sut.count == 5)
    await sut.send(.viewDisappeared)
}
```

- Producer tests are non-flaky (`latestValue` is synchronous). Consumer tests are inherently async — use `waitUntil`, never assert immediately after a `send`.
- To drive the full round trip, send `.task` (establish subscription) then the producing Action, then `waitUntil` on the resulting state.

### Parent action delegation test

When testing that a Store correctly delegates to its parent via `action`, use `TestStore`:

```swift
@Test
func send_closeButtonTapped_parentReceivesAction() async {
    let sut = TestStore { action in
        ChildStore(action: action)
    }
    await sut.send(.closeButtonTapped)
    sut.receive {
        if case .closeButtonTapped = $0 { true } else { false }
    }
}
```

---

## Key Rules

1. **`action: { _ in }`** — always pass a no-op `action` closure when the parent delegation is not under test
2. **Inject pre-set initial state** via `init` parameters for precondition setup (avoid reaching in and mutating after init)
3. **One assertion per test** — each test verifies a single outcome; write separate tests for each scenario
4. **Use `OSAllocatedUnfairLock` for `@Sendable` capture** — never use `var` captured in closures directly
5. **`@testable import Model`** is required to access internal types in tests

---

Generate complete, compilable test files with proper imports. Cover both happy-path and edge-case scenarios. Name every test following the convention above.
