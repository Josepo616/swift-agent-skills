# Swift Agent Skills

A collection of Claude Code agent skills for Swift and SwiftUI development.

## Installation

### Option 1: Plugin Marketplace (recommended)

Add this repository as a plugin marketplace, then install the skills you need:

```
/plugin marketplace add Josepo616/swift-agent-skills
```

Install a specific skill:

```
/plugin install swift-naming-conventions@swift-agent-skills
```

### Option 2: Manual

Copy any skill file directly into your project or personal skills directory:

**Project scope** (available only in the current project):

```
.claude/skills/<skill-name>/SKILL.md
```

**Personal scope** (available across all your projects):

```
~/.claude/skills/<skill-name>/SKILL.md
```

### Available Plugins

| Plugin Name | Skill |
|---|---|
| `swift-naming-conventions` | Naming Conventions |
| `swift-observable-architecture` | Observable Architecture |
| `swift-concurrency` | Swift Concurrency |
| `swift-project-structure` | Project Structure |
| `swift-protocol-oriented` | Protocol-Oriented Programming |
| `swift-error-handling` | Error Handling |
| `swift-dependency-injection` | Dependency Injection |
| `swift-swiftui-performance` | SwiftUI Performance |
| `swift-access-control` | Access Control |
| `swift-testing` | Swift Testing |
| `swift-accessibility` | Accessibility |
| `swift-previews` | Previews |
| `swift-codable-data-modeling` | Codable & Data Modeling |

## Skills

| # | Skill | Description |
|---|-------|-------------|
| 01 | Naming Conventions | Apply Swift naming conventions and API design guidelines |
| 02 | Observable Architecture | Use @Observable and modern MVVM architecture patterns in SwiftUI |
| 03 | Swift Concurrency | Apply Swift concurrency patterns with async/await, Actors, and Sendable |
| 04 | Project Structure | Organize SwiftUI project structure and file layout |
| 05 | Protocol-Oriented Programming | Apply protocol-oriented programming patterns in Swift |
| 06 | Error Handling | Apply Swift error handling best practices |
| 07 | Dependency Injection | Implement dependency injection patterns in SwiftUI |
| 08 | SwiftUI Performance | Optimize SwiftUI view performance |
| 09 | Access Control | Apply Swift access control levels and code organization |
| 10 | Swift Testing | Write tests using the Swift Testing framework |
| 11 | Accessibility | Implement SwiftUI accessibility best practices |
| 12 | Previews | Set up and use SwiftUI previews effectively |
| 13 | Codable & Data Modeling | Design data models with Codable and JSON parsing |

## License

See [LICENSE](LICENSE) for details.
