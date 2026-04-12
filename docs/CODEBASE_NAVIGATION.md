# Peekaboo Codebase Navigation Guide

## Overview

Peekaboo is a macOS automation tool that combines:
1. **Screen capture** - pixel-accurate screenshots with Retina scaling
2. **AI analysis** - natural-language agent with multi-provider support
3. **GUI automation** - complete control via Accessibility APIs

## Directory Structure

```
Peekaboo/
├── 📁 Apps/                          # Consumer applications
│   ├── CLI/                         # Command-line interface (SwiftPM)
│   ├── Mac/                         # Native macOS app (Xcode)
│   ├── Playground/                  # Testing playground
│   └── PeekabooInspector/           # Inspection tooling
│
├── 📁 Core/                         # Shared libraries
│   ├── PeekabooCore/                # Umbrella module
│   │   ├── PeekabooAgentRuntime/    # MCP + Agent service
│   │   ├── PeekabooAutomation/      # Automation services
│   │   ├── PeekabooBridge/          # Socket bridge for remote execution
│   │   └── PeekabooCore/            # Core logic & service locator
│   │
│   ├── PeekabooAutomationKit/       # Automation APIs & models
│   ├── PeekabooFoundation/          # Foundation utilities
│   ├── PeekabooProtocols/           # Protocol definitions
│   └── PeekabooVisualizer/          # Visual feedback layer
│
├── 📁 AXorcist/                     # AX automation (git submodule)
├── 📁 Commander/                    # CLI parsing (git submodule)
├── 📁 Tachikoma/                    # AI providers/MCP (git submodule)
├── 📁 TauTUI/                       # TUI components (git submodule)
├── 📁 Swiftdansi/                   # ANSI styling (git submodule)
│
├── 📁 docs/                         # Documentation
├── 📁 scripts/                      # Build & utility scripts
├── 📁 Examples/                     # Example automation scripts
└── 📁 assets/                       # Marketing assets
```

## Key Modules Deep Dive

### Apps/CLI - Command Line Interface

**Entry Points:**
- `Apps/CLI/main.swift` - Primary CLI entry
- `Apps/CLI/Sources/peekaboo/main.swift` - Legacy/alternative entry
- `Apps/CLI/Sources/PeekabooExec/main.swift` - Execution helper

**Structure:**
```
Apps/CLI/Sources/PeekabooCLI/
├── CLI/                  # Core CLI infrastructure
├── Commands/             # Command implementations
│   ├── Agent/           # Autonomous agent loop
│   ├── AI/              # AI provider integrations
│   ├── Base/            # Base command classes
│   ├── Core/            # Core automation commands
│   ├── Interaction/     # Input commands (click, type, etc.)
│   ├── MCP/             # MCP server commands
│   ├── Shared/          # Shared utilities
│   └── System/          # System commands (app, window, etc.)
├── Helpers/             # Command helpers
└── Logging/             # Logging infrastructure
```

**Common Commands Location:**
- `click`, `type`, `scroll` → `Commands/Interaction/`
- `see`, `image`, `capture` → `Commands/Core/`
- `app`, `window`, `menu` → `Commands/System/`
- `agent` → `Commands/Agent/`
- `mcp` → `Commands/MCP/`

### Core/PeekabooCore - Service Layer

**PeekabooAutomation:**
```
Services/
├── UIAutomationService.swift      # Main orchestrator
├── ScreenCaptureService.swift     # Screenshots
├── ApplicationService.swift       # App lifecycle
├── WindowManagementService.swift  # Window manipulation
├── MenuService.swift              # Menu interactions
├── ClickService.swift             # Mouse clicks
├── TypeService.swift              # Keyboard input
├── DialogService.swift            # System dialogs
└── DockService.swift              # Dock interactions
```

**PeekabooAgentRuntime:**
```
├── PeekabooAgentService.swift     # Agent loop
├── ToolRegistry.swift             # Tool registration
├── MCP/                           # MCP server integration
└── Tools/                         # Agent tools
```

**PeekabooBridge:**
```
├── BridgeClient.swift             # UNIX socket client
├── BridgeHost.swift               # Socket server
├── Messages/                      # Bridge protocol
└── Security/                      # Code signature validation
```

### Core Submodules

**AXorcist** - Accessibility automation:
- Low-level AX API wrappers
- Element traversal
- Attribute reading/writing

**Tachikoma** - AI providers:
- `AIModelProvider` - model registry
- `AIModelFactory` - model creation
- Providers: OpenAI, Anthropic, Grok, Google, Ollama
- MCP SDK integration

**Commander** - CLI framework:
- Command parsing
- Argument handling
- Help generation

## Data Flow Examples

### Example 1: `peekaboo see --annotate`

```
CLI Command (Apps/CLI/Sources/PeekabooCLI/Commands/Core/SeeCommand.swift)
    ↓
UIAutomationService.capture() → ScreenCaptureService
    ↓
ElementDetectionService.detect() → AXorcist traversal
    ↓
SnapshotManager.store() → ~/.peekaboo/snapshots/<id>/
    ↓
VisualizerClient.show() → Overlay annotations
```

**Files to check:**
- `SeeCommand.swift` - command logic
- `ScreenCaptureService.swift` - capture
- `ElementDetectionService.swift` - AI detection
- `SnapshotManager.swift` - persistence

### Example 2: `peekaboo click --on "Submit"`

```
CLI Command (Apps/CLI/Sources/PeekabooCLI/Commands/Interaction/ClickCommand.swift)
    ↓
SnapshotManager.getElement() → load from cache
    ↓
ElementResolver.resolve("Submit") → match element
    ↓
UIAutomationService.click() → ClickService
    ↓
AXorcist.click(element) → macOS Accessibility API
    ↓
VisualizerClient.showClickFeedback()
```

**Files to check:**
- `ClickCommand.swift` - command logic
- `ElementResolver.swift` - ID/label matching
- `ClickService.swift` - click implementation
- `UIAutomationService.swift` - orchestration

### Example 3: `peekaboo agent "Open Safari"`

```
CLI Command (Apps/CLI/Sources/PeekabooCLI/Commands/Agent/AgentCommand.swift)
    ↓
PeekabooAgentService.run()
    ↓
Tachikoma.getModel() → AI provider
    ↓
Agent loop: interpret → tool call → execute
    ↓
ToolRegistry.execute(tool: "app_launch", args: {...})
    ↓
ApplicationService.launch() → NSWorkspace
```

**Files to check:**
- `AgentCommand.swift` - command entry
- `PeekabooAgentService.swift` - agent loop
- `ToolRegistry.swift` - tool dispatch
- `Tools/AppLaunchTool.swift` - specific tool

## Finding Code

### By Feature

**Screen capture:**
```bash
# Search terms
grep -r "captureWindow\|captureScreen" Core/ Apps/CLI/Sources/
# Key files
Core/PeekabooCore/Sources/PeekabooAutomation/Services/ScreenCaptureService.swift
```

**Element detection:**
```bash
grep -r "detectElements\|ElementDetection" Core/ Apps/CLI/Sources/
# Key files
Core/PeekabooCore/Sources/PeekabooAutomation/Services/ElementDetectionService.swift
```

**Menu interactions:**
```bash
grep -r "clickMenuItem\|MenuService" Core/ Apps/CLI/Sources/
# Key files
Core/PeekabooCore/Sources/PeekabooAutomation/Services/MenuService.swift
```

### By Command

**Find command implementation:**
```bash
# Find command files
find Apps/CLI/Sources -name "*Command.swift" | grep -i click
# Result: Apps/CLI/Sources/PeekabooCLI/Commands/Interaction/ClickCommand.swift
```

**Find service implementation:**
```bash
find Core -name "*Service.swift" | grep -i click
# Result: Core/PeekabooCore/Sources/PeekabooAutomation/Services/ClickService.swift
```

## Understanding Dependencies

### Module Dependencies

```
PeekabooCore (umbrella)
    ↓
PeekabooAgentRuntime
    ↓
PeekabooAutomation + PeekabooVisualizer
    ↓
PeekabooAutomationKit + PeekabooBridge
    ↓
PeekabooFoundation + PeekabooProtocols + AXorcist
```

### External Dependencies

**Package.swift:**
- `AXorcist` - Accessibility automation
- `swift-algorithms` - Swift Algorithms library

**Submodules:**
- `Tachikoma` - AI providers (OpenAI, Anthropic, etc.)
- `Commander` - CLI framework
- `TauTUI` - Terminal UI
- `Swiftdansi` - ANSI styling

## Testing Structure

### Unit Tests
```
Apps/CLI/Tests/
├── PeekabooCLITests/
└── PeekabooAutomationTests/
```

### Test Commands
```bash
# Safe tests (no permissions)
pnpm run test:safe

# Full tests (needs permissions)
pnpm run test:automation
pnpm run test:all
```

## Build System

### Swift Package Manager
- Root `Package.swift` defines library targets
- Apps/CLI has its own `Package.swift` for CLI binary
- Apps/Mac uses Xcode project

### Build Scripts
```bash
pnpm run build:cli              # Debug CLI
pnpm run build:swift            # Debug all Swift
pnpm run build:swift:all        # Release universal
pnpm run poltergeist:haunt      # Watch mode
```

## Common Patterns

### Service Locator Pattern
```swift
let services = PeekabooServices()
services.installAgentRuntimeDefaults()
let automation = services.automation
let screenCapture = services.screenCapture
```

### MainActor Isolation
All UI automation runs on MainActor:
```swift
@MainActor
public final class UIAutomationService {
    // All methods are main-actor isolated
}
```

### Dependency Injection (Tachikoma)
```swift
// Recommended approach
let provider = try AIConfiguration.fromEnvironment()
let model = try provider.getModel("gpt-5.1")
```

### Snapshot-Based Workflow
```swift
// Create snapshot
let snapshotId = try await snapshotManager.createSnapshot()

// Store detection
try await snapshotManager.storeDetectionResult(...)

// Retrieve element
let element = try await snapshotManager.getElement(snapshotId: ..., elementId: ...)
```

## Quick Navigation Tips

### Using grep
```bash
# Find function definition
grep -r "func captureWindow" Core/

# Find type usage
grep -r "ScreenCaptureService" Apps/CLI/Sources/

# Find protocol conformances
grep -r ": ScreenCaptureServiceProtocol" Core/
```

### Using find
```bash
# Find all Swift files for a service
find Core -name "*Service*.swift"

# Find test files
find Apps/CLI/Tests -name "*Tests.swift"
```

### Using AGENTS.md
Always check `AGENTS.md` and `TOOLS.MD` in `~/Projects/agent-scripts/` for additional context if available.

## Documentation Locations

- `README.md` - Quick start
- `CHANGELOG.md` - Recent changes
- `docs/ARCHITECTURE.md` - System overview
- `docs/service-api-reference.md` - API docs
- `docs/commands/*.md` - Command-specific docs
- `docs/permissions.md` - Permission guide
- `docs/building.md` - Build instructions
