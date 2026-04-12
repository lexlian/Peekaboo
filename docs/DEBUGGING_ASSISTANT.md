# Peekaboo Debugging Assistant

## Quick Reference for Debugging

### Permission Issues (Most Common)
```bash
# Check permissions first
peekaboo permissions status

# Grant if needed
peekaboo permissions grant

# Required permissions:
# 1. Screen Recording - System Settings → Privacy → Screen & System Audio Recording
# 2. Accessibility - System Settings → Privacy → Accessibility
# Enable for: Terminal, your IDE, or whatever runs peekaboo
```

### Build & Run Locally
```bash
# Quick debug build
pnpm run build:swift

# Watch mode (auto-rebuild on changes)
pnpm run poltergeist:haunt

# Check watcher status
pnpm run poltergeist:status

# Run with local build
polter peekaboo <command> <args>

# Stop watcher
pnpm run poltergeist:rest
```

### Debug Workflow

1. **Reproduce the issue**
   ```bash
   peekaboo <command> --verbose
   ```

2. **Check logs and errors**
   - Look for permission errors
   - Check timeout messages
   - Note AX (Accessibility) failures

3. **Common fixes**
   - Restart affected apps after granting permissions
   - Re-run `peekaboo permissions grant`
   - Check focus: `peekaboo app focus <app>`

### Module Structure

```
Peekaboo/
├── Apps/
│   ├── CLI/                    # Command-line tool
│   │   ├── Sources/
│   │   │   ├── peekaboo/       # Main entry
│   │   │   ├── PeekabooCLI/    # CLI logic
│   │   │   └── PeekabooExec/   # Exec helpers
│   │   └── Tests/
│   ├── Mac/                    # macOS app
│   │   └── Peekaboo/
│   └── Playground/             # Testing playground
├── Core/
│   ├── PeekabooCore/           # Umbrella module
│   │   ├── PeekabooAgentRuntime/  # MCP + agent service
│   │   ├── PeekabooAutomation/    # Automation services
│   │   ├── PeekabooBridge/        # Socket bridge
│   │   └── PeekabooCore/          # Core logic
│   ├── PeekabooAutomationKit/  # Automation APIs
│   ├── PeekabooFoundation/     # Foundation utilities
│   ├── PeekabooProtocols/      # Protocol definitions
│   └── PeekabooVisualizer/     # Visual feedback
├── AXorcist/                   # AX automation (submodule)
├── Commander/                  # CLI parsing (submodule)
├── Tachikoma/                  # AI providers (submodule)
└── TauTUI/                     # TUI components (submodule)
```

### Key Architecture Concepts

**Service Architecture:**
- `UIAutomationService` - orchestrates all automation
- `ScreenCaptureService` - screenshots (CGWindow + ScreenCaptureKit)
- `ApplicationService` - app lifecycle
- `WindowManagementService` - window manipulation
- `MenuService` - menu bar interactions
- `ClickService`, `TypeService`, etc. - input primitives

**Snapshot System:**
- `peekaboo see` creates snapshots with UI element maps
- Follow-up commands (`click`, `type`, `scroll`) reuse snapshots
- Stored in `~/.peekaboo/snapshots/<id>/`
- Includes: `snapshot.json`, `raw.png`, optionally `annotated.png`

**Bridge Host:**
- Privileged automation runs in signed host (Peekaboo.app)
- CLI connects via UNIX socket
- Avoids repeated permission prompts
- Debug: `peekaboo bridge status`

**MCP Server:**
- Runs as `peekaboo mcp` (default: serve)
- Provides tools to Claude Desktop, Cursor, etc.
- Uses same tooling as CLI

### Debug Tools

**Verbose Output:**
```bash
peekaboo <command> --verbose
peekaboo see --annotate --verbose
```

**Bridge Diagnostics:**
```bash
peekaboo bridge status
```

**List State:**
```bash
peekaboo list apps
peekaboo list windows
peekaboo list permissions
```

**Test Commands:**
```bash
# Safe tests (no automation permissions needed)
pnpm run test:safe

# Full automation tests (requires permissions)
pnpm run test:automation

# Integration tests
pnpm run tachikoma:test:integration
```

### Common Issues & Solutions

#### "Permission denied" errors
```bash
peekaboo permissions status
peekaboo permissions grant
# Then restart terminal/IDE
```

#### Element not found
```bash
# Use annotate mode to see element IDs
peekaboo see --annotate

# Try coordinate-based fallback
peekaboo click --coords "x,y"

# Check if app is focused
peekaboo app focus <app>
```

#### Slow automation
- Add delays: `peekaboo sleep --duration 500`
- Check `--timeout-sec` flags
- Verify both permissions granted
- Use `--quiet-ms` / `--heartbeat-sec` for capture commands

#### Window targeting issues
```bash
# List windows with IDs
peekaboo list windows --app <app>

# Target by ID instead of title
peekaboo window focus --window-id <id>
```

#### Menu bar interactions
```bash
# List menu extras
peekaboo menubar list

# Click with verification
peekaboo menubar click "Item Name" --verify
```

### Swift-Specific Debugging

**Compiler crashes:**
```bash
# Clean build
rm -rf .build
pnpm run build:swift
```

**Swift 6 concurrency issues:**
- Check MainActor isolation
- Review strict concurrency settings in Package.swift
- Use `@MainActor` for UI automation code

**Performance profiling:**
- Enable warnings: `-warn-long-function-bodies=50`
- Check logs for AX traversal timeouts
- Monitor snapshot cache usage

### Git Submodules

When updating submodules:
```bash
# Update in their home repos first
cd AXorcist && git pull
cd ../Tachikoma && git pull
# Then bump pointers in main repo
git submodule update --remote
```

### Configuration Files

- `~/.peekaboo/config.json` - main config
- `~/.tachikoma/credentials` - AI provider keys
- `poltergeist.config.json` - local dev watcher
- `.swiftlint.yml` - linting rules
- `.swiftformat` - formatting rules

### Environment Variables

```bash
PEEKABOO_AI_PROVIDERS="openai/gpt-5.1,anthropic/claude-opus-4"
PEEKABOO_USE_MODERN_CAPTURE=1
PEEKABOO_MENUBAR_OCR_VERIFY=0
PEEKABOO_INCLUDE_AUTOMATION_TESTS=true
```

### Getting Help

**Documentation:**
- `docs/commands/` - individual command docs
- `docs/ARCHITECTURE.md` - system overview
- `docs/service-api-reference.md` - API reference
- `docs/permissions.md` - permission guide

**When asking for help, include:**
1. macOS version
2. Peekaboo version (`peekaboo --version`)
3. Command that failed (with flags)
4. Full error output
5. Permission status
6. What you've tried
