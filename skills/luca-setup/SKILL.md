---
name: luca-setup
description: Guides project scaffolding for LUCA-based Xcode projects — installing prerequisites, running the luca CLI, and understanding the generated project structure.
---

You are an expert in setting up new **LUCA architecture** projects. Guide the user through installing the required tools and generating a new LUCA-based Xcode project.

---

## Requirements

| Tool                                              | Version |
| ------------------------------------------------- | ------- |
| Xcode                                             | 26.0+   |
| iOS deployment target                             | 17.0+   |
| macOS deployment target                           | 14.0+   |
| Swift                                             | 6.2     |
| [XcodeGen](https://github.com/yonaskolb/XcodeGen) | 2.44.1+ |

---

## Step 1 — Install XcodeGen

The `luca` CLI depends on XcodeGen to generate the `.xcodeproj` file.

```sh
brew install xcodegen
```

---

## Step 2 — Install the `luca` CLI

**Option A: Homebrew (recommended)**

```sh
brew tap kyome22/tap
brew install luca
```

**Option B: Swift Package Manager (build from source)**

```sh
git clone https://github.com/kyome22/LUCA.git
cd LUCA
swift run luca
```

---

## Step 3 — Generate a New Project

```sh
luca --name <name> --organization-id <organization-id> [--platform <platform>] [--path <path>]
```

**Options**

| Short  | Long                | Description                                                        |
| ------ | ------------------- | ------------------------------------------------------------------ |
| `-n`   | `--name`            | Project name (e.g., `MyApp`)                                       |
| `-o`   | `--organization-id` | Organization identifier (e.g., `com.example`)                      |
| (none) | `--platform`        | Target platform: `iOS` or `macOS` (default: `iOS`)                 |
| `-p`   | `--path`            | Directory to create the project in (defaults to current directory) |

**Example**

```sh
# iOS (default)
luca --name MyApp --organization-id com.example --path ~/Developer

# macOS
luca --name MyApp --organization-id com.example --platform macOS --path ~/Developer
```

---

## Generated Project Structure

```
.
├── LocalPackage/
│   ├── Package.swift
│   ├── Sources/
│   │   ├── DataSource/
│   │   │   ├── Dependencies/
│   │   │   │   └── AppStateClient.swift
│   │   │   ├── Entities/
│   │   │   │   └── AppState.swift
│   │   │   ├── Extensions/
│   │   │   ├── Repositories/
│   │   │   └── DependencyClient.swift
│   │   ├── Model/
│   │   │   ├── Extensions/
│   │   │   ├── Services/
│   │   │   ├── Stores/
│   │   │   ├── AppDelegate.swift    ← for app lifecycle events
│   │   │   ├── AppDependencies.swift
│   │   │   └── Composable.swift
│   │   └── UserInterface/
│   │       ├── Extensions/
│   │       ├── Resources/
│   │       ├── Scenes/
│   │       └── Views/
│   └── Tests/
│       └── ModelTests/
│           ├── TestStore.swift    ← testing utility
│           ├── ServiceTests/
│           └── StoreTests/
├── MyApp/
│   └── MyAppApp.swift
└── MyApp.xcodeproj
```

---

## Next Steps

Once the project is open in Xcode:

- Use `/luca-arch` to understand the LUCA architecture and each layer's responsibilities
- Use `/luca-impl` to implement features (DataSource → Model → UserInterface)
- Use `/luca-test` to write unit tests for your Services and Stores
