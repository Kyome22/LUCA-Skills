---
name: luca-arch
description: Explains the LUCA architecture — its three-layer structure, key components, data flow, and patterns for SwiftUI app development.
---

You are an expert on the **LUCA architecture** for SwiftUI applications. Answer the user's questions about LUCA clearly and accurately using the knowledge below.

---

## What is LUCA?

**LUCA** (Layered, Unidirectional Data Flow, Composable, Architecture) is a lightweight SwiftUI architecture optimized for the SwiftUI × Observation era. It is inspired by TCA but implemented using only Apple-native frameworks (no third-party dependencies).

> "Clean layers, Clear flow, Composable design"

**Design goals:**

- Suitable for solo or early-stage app development
- Only Apple-native frameworks and apple OSS libraries
- Xcode 26.0+, Swift 6.2+, iOS 18.0+ / macOS 15.0+

---

## Three-Layer Structure

```
┌───────────────────┐
│   UserInterface   │  ← UI provision and event handling
├───────────────────┤
│       Model       │  ← Business logic and state management
├───────────────────┤
│    DataSource     │  ← Data access and infrastructure abstraction
└───────────────────┘
```

Each layer is delivered as a separate Swift Package module (`DataSource`, `Model`, `UserInterface`) in a local package (`LocalPackage`). Dependencies flow only downward: `UserInterface → Model → DataSource`.

---

## Foundational principles (the "why" — reason from these, not from examples)

Everything else in LUCA is a consequence of these. Internalize them and most concrete rules become derivable, so you rarely need a worked example to know the right shape.

1. **Declarative UI is the bedrock.** LUCA exists *because* SwiftUI is declarative: a View is a pure function of state, which is what lets the View layer be a **separate module** that depends on logic but is never depended upon. Module-level separation of UI from logic is the whole starting point — not a stylistic preference.
2. **Keep UserInterface a pure declarative module; push imperative glue below it.** AppDelegate, AppKit, `NSWindow`/`NSPanel` ownership, and lifecycle wiring live in `Model` (or are wrapped as clients in `DataSource`) — never in `UserInterface`. A bridge that drags an `AppDelegate` or AppKit object up into the UI module **breaks the separation** (e.g. letting an `NSPanel` bridge pull an `AppDelegate` into the UI layer). When a window/panel needs `NSWindow`-level control that native SwiftUI scenes can't express, wrap it as a declarative `Scene` instead — with a small wrapper of your own, or a library of that kind if you choose to add one — so the UI module stays pure SwiftUI and the layer boundary holds. LUCA itself depends on nothing beyond Apple / swift-lang, so treat any such library as optional, never required.
3. **Logic is split into `Model` and `DataSource` chiefly for testability.** `Model` holds the logic you want to unit-test with live behavior; `DataSource` isolates everything that resists testing — the uncontrolled, side-effecting world — behind `DependencyClient`s.
4. **`DependencyClient` is the core of LUCA's testability.** Its job is to narrow the boundary with uncontrolled side-effecting APIs (system frameworks, clock, randomness, I/O, third-party SDKs) to the **thinnest possible seam**. The thinner that seam, the more of your real logic runs under test against live behavior, and the more credible your unit tests are as a guarantee of correctness. This is *why* a client must stay a thin 1:1 wrapper and never absorb logic: any logic pulled inside the seam is logic you can no longer test. Maximizing tested logic by minimizing the un-mockable surface is the point of the whole DataSource/Model split.

---

## Standard File Layout

```
LocalPackage/
├── Package.swift
├── Sources/
│   ├── DataSource/
│   │   ├── Dependencies/          ← DependencyClient implementations
│   │   │   └── AppStateClient.swift
│   │   ├── Entities/              ← Data models (struct/enum)
│   │   │   ├── AppState.swift
│   │   │   └── AsyncStreamBundle.swift  ← reactive state stream
│   │   ├── Extensions/
│   │   ├── Repositories/          ← Data read/write access
│   │   └── DependencyClient.swift ← Base protocol
│   ├── Model/
│   │   ├── Extensions/
│   │   ├── Services/              ← Stateless business logic
│   │   ├── Stores/                ← State management + event handling
│   │   ├── AppDelegate.swift      ← (optional) App lifecycle
│   │   ├── AppDependencies.swift  ← Dependency container + EnvironmentValues
│   │   └── Composable.swift       ← Store protocol
│   └── UserInterface/
│       ├── Extensions/
│       ├── Resources/             ← Asset/String catalogs (ONLY here)
│       ├── Scenes/                ← Scene definitions
│       └── Views/                 ← SwiftUI views
└── Tests/
    └── ModelTests/
        ├── TestStore.swift   ← testing utility
        ├── ServiceTests/
        └── StoreTests/
```

---

## Key Components

### `DependencyClient` (DataSource)

A protocol that all dependency wrappers conform to. Provides `liveValue` (production) and `testValue` (testing) static instances.

```swift
public protocol DependencyClient: Sendable {
    static var liveValue: Self { get }
    static var testValue: Self { get }
}

public func testDependency<D: DependencyClient>(of type: D.Type, injection: (inout D) -> Void) -> D {
    var d = type.testValue
    injection(&d)
    return d
}
```

A DependencyClient's single purpose is to be a **test seam** (`liveValue`/`testValue`) over a side effect
outside your control — it is *not* a utility, convenience facade, or logic layer (the *why* is foundational
principle #4 above). The operational rules that follow from that:

- **Each closure is a thin 1:1 wrapper of one underlying call** — same parameters, same return, `liveValue`
  delegating straight to the API and `testValue` an inert stub. It adds *nothing* but the seam.
- **No composition, no convenience, no state.** Never bundle several calls into one "helpful" operation,
  hardcode arguments, or hold a stored property / stream-producing box. Logic that combines or sequences
  effects lives in the Service/Store that *calls* the client — testable precisely because the client beneath
  it is mockable. (A stateful "box" inside a client is the classic anti-pattern: it pulls untested
  orchestration back across the seam.)
- **The one exception** is `AppStateClient` (stateful by design — it owns the lock); everything else stays
  stateless and thin.
- **So the layer is mechanical, not designed:** translate each side-effecting call site into one closure
  rather than reasoning client-by-client. (Swift closures can't be generic, so a few APIs can't be wrapped
  perfectly — wrap the concrete surface you call and move on; the principle is unchanged.)

**When to wrap an API:** any side effect outside your control — `Date()`/`UUID()`, file I/O, `UserDefaults`,
`NSWorkspace`/`NSApp`, global event monitors, any third-party SDK. Pure value transformations
and your own logic are *not* clients.

### `AppState` / `AppStateClient` (DataSource)

`AppState` centralizes all app-wide shared state. `AppStateClient` owns an `OSAllocatedUnfairLock<AppState>` and delegates `withLock` to it, so the **entire body runs under the lock** — read-modify-write is atomic.

```swift
import os

public struct AppStateClient: DependencyClient {
    private let lock: OSAllocatedUnfairLock<AppState>

    private init(lock: OSAllocatedUnfairLock<AppState>) {
        self.lock = lock
    }

    public func withLock<R: Sendable>(_ body: @Sendable (inout AppState) throws -> R) rethrows -> R {
        try lock.withLock(body)
    }

    public static let liveValue = Self(lock: .init(initialState: .init()))
    public static let testValue = Self(lock: .init(initialState: .init()))

    public static func testDependency(_ appState: OSAllocatedUnfairLock<AppState>) -> Self {
        Self(lock: appState)
    }
}
```

> The lock is non-recursive — never call `withLock` (or `send`, below) again inside a `withLock` body.
>
> **The lock is pluggable.** `OSAllocatedUnfairLock` (Apple-native, no extra dependency) is the LUCA default. Any type with equivalent `withLock(_:)` semantics works just as well — Kyome's own apps use `Kyome22/AllocatedUnfairLock`, but that is a personal preference, not a LUCA requirement.

### `AsyncStreamBundle` & state subscription (DataSource)

To let multiple Stores **reactively observe** shared state, `AppState` holds `AsyncStreamBundle<T>` fields. Each bundle wraps an `AsyncStream` shared via swift-async-algorithms `share()`, and remembers the last value (`latestValue`) for synchronous read-back.

```swift
import AsyncAlgorithms

public typealias AsyncShareStream<T: Sendable> = Sendable & AsyncSequence<T, AsyncStream<T>.__AsyncSequence_Failure>

public struct AsyncStreamBundle<T>: Sendable where T: Sendable {
    public let stream: any AsyncShareStream<T>
    private let continuation: AsyncStream<T>.Continuation
    public private(set) var latestValue: T? = nil

    public init() {
        let (stream, continuation) = AsyncStream.makeStream(of: T.self, bufferingPolicy: .bufferingNewest(1))
        self.stream = stream.share(bufferingPolicy: .bufferingLatest(1))
        self.continuation = continuation
    }

    public mutating func send(_ value: T) {
        latestValue = value
        continuation.yield(value)
    }
}
```

`AppStateClient` exposes two `send` overloads (the single atomic write path — never call `$0.x.send(...)` directly outside these):

```swift
// push a finished value
public func send<T: Sendable>(
    _ keyPath: any WritableKeyPath<AppState, AsyncStreamBundle<T>> & Sendable, _ value: T
) {
    lock.withLock { $0[keyPath: keyPath].send(value) }
}

// read-modify-write, starting from latestValue (or a default)
public func send<T: Sendable>(
    _ keyPath: any WritableKeyPath<AppState, AsyncStreamBundle<T>> & Sendable,
    default defaultValue: @autoclosure @Sendable () -> T, _ transform: @Sendable (inout T) -> Void
) {
    lock.withLock { appState in
        var value = appState[keyPath: keyPath].latestValue ?? defaultValue()
        transform(&value)
        appState[keyPath: keyPath].send(value)
    }
}
```

**Producer** (a Service) pushes values; **consumer** (a Store) subscribes:

```swift
// Producer: Service
appStateClient.send(\.count, default: 0) { $0 += 1 }

// Consumer: Store, inside reduce(.task)
if let latest = appStateClient.withLock(\.count.latestValue) { count = latest }
task?.cancel()
task = Task { [weak self, appStateClient] in
    let stream = appStateClient.withLock(\.count.stream)
    for await value in stream { self?.count = value }   // Task inherits @MainActor — no await needed
}
// and cancel it in reduce(.onDisappear): task?.cancel(); task = nil
```

`share` does **not** replay past values to a late subscriber — it only delivers elements produced after that subscriber attaches. That is why the Store seeds its current value synchronously from `latestValue` (read under the lock in `reduce(.task)`) before starting `for await`. `bufferingLatest(1)` on the share keeps a slow consumer from back-pressuring the others — it drops older elements instead of suspending the source (the default `.bounded(1)` would suspend).

### `AppDependencies` (Model)

Collects all dependencies and exposes them via `EnvironmentValues` for injection into Stores from Views/Scenes.

```swift
public final class AppDependencies: Sendable {
    public let appStateClient: AppStateClient
    // ... other clients
    static let shared = AppDependencies()
}

extension EnvironmentValues {
    @Entry public var appDependencies = AppDependencies.shared
}
```

### `Composable` (Model)

The protocol all Stores conform to. Defines the unidirectional event loop.

```swift
@MainActor
public protocol Composable: AnyObject {
    associatedtype Action: Sendable
    var action: (Action) async -> Void { get }
    func reduce(_ action: Action) async
}

public extension Composable {
    func reduce(_ action: Action) async {}
    func send(_ action: Action) async {
        await reduce(action)      // 1. handle internally
        await self.action(action) // 2. delegate to parent
    }
}
```

---

## Data Flow

```
View
 │  user interaction / lifecycle event
 ▼
store.send(.someAction)
 │
 ├─► reduce(.someAction)   ← updates @Observable properties
 │
 └─► action(.someAction)   ← delegates to parent Store (if any)
         │
         ▼
      parent.send(.child(.someAction))
```

Views observe Store properties automatically via `@Observable`. There is no manual binding boilerplate.

---

## Service vs Store

|        | Service                               | Store                                |
| ------ | ------------------------------------- | ------------------------------------ |
| Type   | `struct` (or `actor`)                 | `@MainActor @Observable final class` |
| State  | None — stateless                      | Holds view state as `var` properties |
| Role   | Pure business logic / data processing | Orchestrates UI state + events       |
| Access | Instantiated inside Store's `init`    | Held by View as `@State var store`   |

---

## Child Store Delegation

A child Store delegates its events to its parent via the `action` closure:

```swift
// In parent's reduce:
case .openChild:
    child = .init(action: { [weak self] childAction in
        await self?.send(.child(childAction))
    })

case .child(.closeButtonTapped):
    child = nil

// Parent's Action:
enum Action {
    case openChild
    case child(Child.Action)
}
```

If the child needs `AppDependencies`, pass it through the Action:

```swift
case let .openChild(appDependencies):
    child = .init(appDependencies, action: { [weak self] in ... })
```

---

## Type-Safe Navigation (NavigationStack)

Define a `Path` enum inside the parent Store that conforms to `Hashable`:

```swift
public enum Path: Hashable {
    case detail(DetailStore)

    public func hash(into hasher: inout Hasher) {
        hasher.combine(ObjectIdentifier(store))
    }
    public static func ==(lhs: Path, rhs: Path) -> Bool {
        lhs.id == rhs.id
    }
}
```

Use `NavigationStack(path: $store.path)` with `.navigationDestination(for: Parent.Path.self)` in the View.

---

## Resource Placement Rule

Images and localized strings live **only** in `UserInterface/Resources/`. If the Model layer needs string keys, use `String.LocalizationValue`. If a DataSource entity needs a display string/image, add an extension in `UserInterface/Extensions/`.

---

When the user asks a question, answer it using the LUCA concepts above. Provide Swift code examples when helpful. If the user asks how to implement something, suggest using `/luca-impl` for step-by-step implementation guidance.
