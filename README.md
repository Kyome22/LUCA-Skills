# LUCA-Skills

[Claude Code](https://claude.ai/claude-code) skills for developing apps with the [LUCA architecture](https://github.com/kyome22/LUCA).

## What is LUCA?

LUCA (Layered, Unidirectional Data Flow, Composable, Architecture) is a lightweight SwiftUI architecture optimized for the SwiftUI × Observation era. It is inspired by TCA but implemented using only Apple-native frameworks.

## Skills

| Command       | Description                                                                      |
| :------------ | :------------------------------------------------------------------------------- |
| `/luca-arch`  | Explains the LUCA architecture: layers, components, data flow, and patterns      |
| `/luca-setup` | Guides project scaffolding — installing prerequisites and running the `luca` CLI |
| `/luca-impl`  | Implements features across all three layers following LUCA coding rules          |
| `/luca-test`  | Writes unit tests for Services and Stores using Swift Testing                    |

## Installation

Copy the `skills/` directory into your project's `.claude/` folder:

```sh
cp -r skills/* /path/to/your/project/.claude/skills/
```

Or to install globally for all projects:

```sh
cp -r skills/* ~/.claude/skills/
```

## Usage

Invoke a skill from any Claude Code session with its slash command:

```
/luca-arch  → ask questions about the LUCA architecture
/luca-setup → set up a new LUCA project from scratch
/luca-impl  → implement a new feature or screen
/luca-test  → write unit tests for a Store or Service
```

## Requirements

- [Claude Code](https://claude.ai/claude-code)

## Related

- [LUCA](https://github.com/kyome22/LUCA) — CLI tool to scaffold LUCA-based Xcode projects
