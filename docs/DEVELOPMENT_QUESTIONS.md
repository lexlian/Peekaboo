# Peekaboo Development & Debugging Knowledge Gaps

*Living document of questions, unknowns, and areas requiring deeper understanding*

---

## 🔍 Critical Architecture Questions

### 1. Bridge Host Communication

**What needs investigation:**
- [ ] What is the exact message protocol between CLI and Bridge host?
- [ ] How are errors serialized across the socket boundary?
- [ ] What happens when bridge host crashes mid-operation?
- [ ] How does authentication/team ID validation work in detail?
- [ ] What's the retry strategy for socket connection failures?

**Files to examine:**
```
Core/PeekabooCore/Sources/PeekabooBridge/
├── BridgeClient.swift
├── BridgeHost.swift
└── Messages/
```

**Debug approach:**
```bash
peekaboo bridge status --verbose
# Add logging to BridgeClient.send() and BridgeHost.receive()
```

**Why it matters:** Many permission and crash issues (#76, #77, #73) likely stem from bridge communication failures.

---

### 2. Snapshot Lifecycle & Cache Management

**What needs investigation:**
- [ ] When are snapshots invalidated? (window changes, app switches, time-based?)
- [ ] How does the snapshot know if the underlying window has changed?
- [ ] What's the memory footprint of in-memory snapshots?
- [ ] How are snapshot IDs scoped per bundle ID?
- [ ] What happens to snapshots when bridge host restarts?

**Files to examine:**
```
Core/PeekabooCore/Sources/PeekabooCore/
├── SnapshotManager.swift
└── UIAutomationSnapshot.swift

Core/PeekabooCore/Sources/PeekabooAutomation/Services/
└── SnapshotAutomationService.swift
```

**Debug questions:**
```swift
// How is windowID tracked?
// What invalidates the cache?
// Is there a TTL or LRU eviction?
```

**Why it matters:** Issue #63 (coordinate mapping) suggests snapshot → screen coordinate transformation is broken in some cases.

---

### 3. Element Detection & Coordinate Mapping

**What needs investigation:**
- [ ] How does AX element frame (accessibilityFrame) map to screen coordinates?
- [ ] What coordinate space is used? (flipped vs unflipped, display vs window-local)
- [ ] How are Retina displays handled? (2x scaling factors)
- [ ] What happens with multi-monitor setups?
- [ ] How does element detection handle scrolled content?

**Files to examine:**
```
Core/PeekabooCore/Sources/PeekabooAutomation/Services/
├── ElementDetectionService.swift
├── ElementResolver.swift
└── ScreenCaptureService.swift

Core/PeekabooAutomationKit/Sources/PeekabooAutomationKit/
├── Models/UIElement.swift
└── Utils/CoordinateTransform.swift (if exists)
```

**Test cases needed:**
```bash
# Multi-monitor
peekaboo see --screen 2 --annotate
peekaboo click --on "elem_1" --verbose

# Retina vs non-Retina
peekaboo image --retina --path /tmp/test.png
# Check actual pixel dimensions vs expected
```

**Why it matters:** Issue #63 directly points to coordinate mapping bugs. Also affects #66 (retina resolution).

---

### 4. Screen Capture Pipeline

**What needs investigation:**
- [ ] When does it use ScreenCaptureKit vs CGWindowList?
- [ ] What are the failure modes for each approach?
- [ ] How is the "modern capture engine" selected?
- [ ] Why do some windows return black frames? (mentioned in CHANGELOG)
- [ ] How are display bounds calculated for multi-monitor crops?

**Files to examine:**
```
Core/PeekabooCore/Sources/PeekabooAutomation/Services/
├── ScreenCaptureService.swift
└── CaptureEngine/ (if exists)

# Environment variables
PEEKABOO_USE_MODERN_CAPTURE=1
```

**Debug approach:**
```bash
# Force different capture engines
peekaboo image --mode window --app "Safari" --verbose
# Check logs for engine selection
```

**Why it matters:** Issues #67 (blank Chrome window), #66 (retina resolution), and general capture reliability.

---

### 5. AI Provider Integration

**What needs investigation:**
- [ ] How are model configurations loaded and prioritized?
- [ ] Why does analyze tool ignore config? (Issue #70)
- [ ] How are vision models called? (Responses API vs Chat Completions)
- [ ] What's the retry strategy for API failures?
- [ ] How are rate limits handled?
- [ ] What's the normalized coordinate conversion for GLM models? (Issue #60)

**Files to examine:**
```
Core/PeekabooCore/Sources/PeekabooAgentRuntime/
├── PeekabooAgentService.swift
└── Tools/AnalyzeTool.swift

# Tachikoma submodule
Tachikoma/Sources/Tachikoma/
├── Providers/OpenAIProvider.swift
├── AIModelFactory.swift
└── AIConfiguration.swift
```

**Debug questions:**
```swift
// Where is the model selected for analyze tool?
// Is it reading from config.json or env vars?
// How do we override the hardcoded gpt-5.1?
```

**Why it matters:** Issue #70 (ignores config), #60 (GLM coordinates), #58 (proxy support).

---

### 6. Permission System

**What needs investigation:**
- [ ] What permissions are actually required? (Screen Recording, Accessibility, AppleEvents?)
- [ ] How does Tahoe (macOS 26+) differ from Sequoia (macOS 15)?
- [ ] Why do CLI apps fail permissions while .app bundles succeed? (Issue #51)
- [ ] How does the bridge host inherit permissions?
- [ ] What's the permission inheritance chain?

**Files to examine:**
```
Apps/CLI/Sources/PeekabooCLI/Commands/System/
└── PermissionsCommand.swift

Core/PeekabooCore/Sources/PeekabooAutomation/Utils/
└── PermissionChecker.swift (if exists)
```

**Test matrix needed:**
```
macOS Version | Install Method | Permission Status | Works?
--------------|----------------|-------------------|--------
15.x          | Homebrew       | Screen+AX         | ?
15.x          | Source         | Screen+AX         | ?
26.x (Tahoe)  | Homebrew       | Screen+AX         | No (#51)
26.x (Tahoe)  | .app bundle    | Screen+AX         | Maybe
```

**Why it matters:** Issues #51, #52, #73, #77 all relate to permissions.

---

## 🧪 Testing Strategy Gaps

### What's Currently Tested

**Safe tests (no permissions):**
- Unit tests for pure functions
- Model serialization
- Command parsing

**Automation tests (need permissions):**
- `pnpm run test:automation`
- Requires Screen Recording + Accessibility

### What's NOT Well Tested

- [ ] Bridge host communication failures
- [ ] Multi-monitor coordinate transformations
- [ ] Retina vs 1x capture scenarios
- [ ] Permission denial edge cases
- [ ] App switching and focus race conditions
- [ ] Element detection with dynamic content
- [ ] Timeout and retry behaviors
- [ ] Memory leaks in long-running sessions

### Testing Recommendations

```swift
// Add integration tests for:
1. Capture pipeline with known windows
2. Element detection on predictable UI (TestHost app)
3. Bridge socket message round-trip
4. Coordinate transform accuracy (known positions)
5. Permission check coverage across macOS versions
```

---

## ⚡ Performance & Concurrency Questions

### Known Issues

- Swift continuation leaks (Issue #52 on Tahoe)
- Wall-clock timeouts for "single action" (10s default)
- AX traversal limits (depth/node/child caps)
- Snapshot caching (~1.5s per-window)

### Questions to Answer

- [ ] Where are the main bottlenecks? (capture vs detection vs AI)
- [ ] How does parallelization work in element detection?
- [ ] What operations are MainActor-isolated and why?
- [ ] Are there race conditions in snapshot updates?
- [ ] How does the build queue handle rapid file changes?

**Profiling approach:**
```bash
# Enable timing logs
PEEKABOO_VERBOSE_TIMING=1 peekaboo see --annotate

# Check AX traversal time
# Check capture time
# Check AI analysis time
```

**Files to examine:**
```
# Concurrency patterns
grep -r "@MainActor" Core/ Apps/CLI/Sources/
grep -r "Task {" Core/ Apps/CLI/Sources/
grep -r "async let" Core/ Apps/CLI/Sources/
```

---

## 🔧 Development Workflow Questions

### Build System

- [ ] What's cached between builds? (`.build/`, `DerivedData/`)
- [ ] How does Poltergeist determine which targets to rebuild?
- [ ] What's the optimal watch path configuration?
- [ ] How to debug Swift compiler crashes? (mentioned in docs)

### Debug Tools

**What's available:**
```bash
peekaboo --verbose
peekaboo bridge status
polter peekaboo --verbose
```

**What's missing:**
- [ ] Structured logging (JSON output)
- [ ] Trace mode for command execution flow
- [ ] Memory profiling hooks
- [ ] Performance metrics export
- [ ] Replay mode for failed automations

### Recommended Debug Setup

```swift
// In your local development:
1. Enable verbose logging in PeekabooServices initialization
2. Add timestamps to all service calls
3. Log coordinate transformations explicitly
4. Capture AX tree snapshots for debugging
5. Add metrics collection for performance baselines
```

---

## 🐛 Common Failure Mode Analysis

### 1. Permission Denials

**Symptoms:**
- "Permission denied" errors
- Silent failures (no prompt shown)
- Works in .app bundle but not CLI

**Debug checklist:**
```bash
# 1. Check status
peekaboo permissions status

# 2. Grant if needed
peekaboo permissions grant

# 3. Verify in System Settings
# System Settings → Privacy → Screen Recording
# System Settings → Privacy → Accessibility

# 4. Restart terminal/IDE

# 5. Check bridge status
peekaboo bridge status
```

**Root causes to investigate:**
- Code signature validation failures
- Team ID allowlist mismatches
- Bridge host running without permissions
- macOS version differences (Tahoe bug #51)

---

### 2. Element Not Found

**Symptoms:**
- `peekaboo click --on "Submit"` fails
- Element visible in screenshot but not detected

**Debug checklist:**
```bash
# 1. Use annotate mode
peekaboo see --annotate --verbose

# 2. Check element IDs
peekaboo see --json | jq '.data.elements[]'

# 3. Try coordinate fallback
peekaboo click --coords "x,y"

# 4. Verify app focus
peekaboo app focus <app>
peekaboo list windows --app <app>
```

**Root causes to investigate:**
- AX tree traversal limits hit
- Element outside scrollable area
- Dynamic content not captured
- Coordinate space mismatch
- Retina scaling factor wrong

---

### 3. Capture Quality Issues

**Symptoms:**
- Black/blank screenshots
- Wrong window captured
- Low resolution (1x instead of 2x)

**Debug checklist:**
```bash
# 1. List available windows
peekaboo list windows --app "Chrome"

# 2. Try different capture modes
peekaboo image --mode screen --retina
peekaboo image --mode window --app "Safari"

# 3. Check actual file dimensions
file /tmp/capture.png
sips -g pixelWidth -g pixelHeight /tmp/capture.png

# 4. Force capture engine
PEEKABOO_USE_MODERN_CAPTURE=1 peekaboo image ...
```

**Root causes to investigate:**
- ScreenCaptureKit stream failures
- CGWindowList returns stale windows
- Display bounds calculation errors
- Retina scaling not applied
- GPU-rendered window incompatibility

---

### 4. Hangs & Timeouts

**Symptoms:**
- Command never returns
- "Operation timed out" after N seconds
- Swift continuation leaks (Tahoe)

**Debug checklist:**
```bash
# 1. Use verbose mode
peekaboo see --verbose --timeout-sec 30

# 2. Check which step hangs
# Add logging to trace execution

# 3. Profile AX traversal
# Add timing logs to ElementDetectionService

# 4. Monitor bridge socket
lsof -U | grep peekaboo
```

**Root causes to investigate:**
- Infinite AX tree loops (should have limits now)
- Bridge socket deadlocks
- Swift continuation never resumed
- Main thread blocked on async operation
- External API calls without timeouts

---

## 📚 Code Locations to Map

### High-Priority Files to Understand

**Bridge Communication:**
- [ ] `Core/PeekabooCore/Sources/PeekabooBridge/BridgeClient.swift`
- [ ] `Core/PeekabooCore/Sources/PeekabooBridge/BridgeHost.swift`
- [ ] `Core/PeekabooCore/Sources/PeekabooBridge/Messages/` (all message types)

**Snapshot Management:**
- [ ] `Core/PeekabooCore/Sources/PeekabooCore/SnapshotManager.swift`
- [ ] `Core/PeekabooCore/Sources/PeekabooCore/UIAutomationSnapshot.swift`
- [ ] `Core/PeekabooCore/Sources/PeekabooAutomation/Services/SnapshotAutomationService.swift`

**Element Detection:**
- [ ] `Core/PeekabooCore/Sources/PeekabooAutomation/Services/ElementDetectionService.swift`
- [ ] `Core/PeekabooCore/Sources/PeekabooAutomation/Services/ElementResolver.swift`
- [ ] `Core/PeekabooAutomationKit/Sources/PeekabooAutomationKit/Models/UIElement.swift`

**Screen Capture:**
- [ ] `Core/PeekabooCore/Sources/PeekabooAutomation/Services/ScreenCaptureService.swift`
- [ ] Check for `CaptureEngine/` subdirectory
- [ ] `Core/PeekabooAutomationKit/Sources/PeekabooAutomationKit/Models/CaptureTarget.swift`

**AI Integration:**
- [ ] `Core/PeekabooCore/Sources/PeekabooAgentRuntime/PeekabooAgentService.swift`
- [ ] `Core/PeekabooCore/Sources/PeekabooAgentRuntime/Tools/` (all tool implementations)
- [ ] `Tachikoma/Sources/Tachikoma/Providers/` (OpenAI, Anthropic, etc.)

**Service Orchestration:**
- [ ] `Core/PeekabooCore/Sources/PeekabooCore/PeekabooServices.swift`
- [ ] `Core/PeekabooCore/Sources/PeekabooAutomation/Services/UIAutomationService.swift`

---

## 🎯 Recommended Investigation Order

### Phase 1: Foundation (Week 1)
1. Bridge communication protocol
2. Snapshot lifecycle
3. Permission system across macOS versions

### Phase 2: Core Automation (Week 2)
4. Element detection & coordinate mapping
5. Screen capture pipeline
6. Service orchestration

### Phase 3: AI Integration (Week 3)
7. AI provider configuration loading
8. Tool execution flow
9. Model selection and fallback

### Phase 4: Edge Cases (Week 4)
10. Multi-monitor setups
11. Retina scaling
12. Performance profiling
13. Memory leak detection

---

## 🛠️ Debug Tools to Build

### Missing Infrastructure

**1. Structured Logging**
```swift
// Add JSON logging mode
peekaboo see --log-format json --output /tmp/trace.json
// Enables programmatic analysis of execution flow
```

**2. Replay Mode**
```bash
# Record automation session
peekaboo agent "..." --record /tmp/session.json

# Replay for debugging
peekaboo replay /tmp/session.json --step-by-step
```

**3. AX Tree Inspector**
```bash
# Dump AX tree for debugging
peekaboo inspect --app "Safari" --output /tmp/ax-tree.json
```

**4. Coordinate Tester**
```bash
# Test coordinate transformations
peekaboo test-coords --element "elem_1" --snapshot "abc123"
# Shows all coordinate spaces and transformations
```

**5. Performance Profiler**
```bash
# Profile command execution
peekaboo see --profile --output /tmp/profile.json
# Breaks down time by: capture, detection, AI, serialization
```

---

## 📋 Questions for Project Maintainers

1. **Bridge Protocol**: Is there documentation for the socket message format?
2. **Snapshot Invalidation**: What triggers snapshot cache invalidation?
3. **Coordinate Spaces**: What's the canonical coordinate space used throughout?
4. **Capture Engines**: When to use ScreenCaptureKit vs CGWindowList?
5. **AI Config**: Why does analyze tool ignore configuration? (Issue #70)
6. **Tahoe Compatibility**: What's different about Tahoe permissions? (Issue #51, #52)
7. **Testing Strategy**: What's the approach for testing without flaky UI tests?
8. **Performance Targets**: What are acceptable latency budgets for operations?
9. **Memory Management**: Are there known memory leaks to watch for?
10. **Roadmap**: What are the priority areas for contribution?

---

## 📖 Related Documentation to Create

Consider adding these docs to the repo:

- [ ] `docs/bridge-protocol.md` - Socket message format reference
- [ ] `docs/coordinate-systems.md` - Coordinate space transformations
- [ ] `docs/snapshot-lifecycle.md` - When snapshots are created/invalidated
- [ ] `docs/capture-engines.md` - ScreenCaptureKit vs CGWindowList decision tree
- [ ] `docs/debugging-cookbook.md` - Common debug scenarios with examples
- [ ] `docs/performance-benchmarks.md` - Expected latencies and throughput
- [ ] `docs/testing-strategy.md` - How to test automation code effectively

---

## Summary

This document identifies **10 major knowledge areas** requiring investigation:

1. Bridge host communication
2. Snapshot lifecycle
3. Element detection & coordinate mapping
4. Screen capture pipeline
5. AI provider integration
6. Permission system
7. Testing gaps
8. Performance & concurrency
9. Development workflow
10. Common failure modes

**Immediate priorities** based on open issues:
- Coordinate mapping (#63) → Investigate #3
- Retina capture (#66) → Investigate #4
- AI config ignored (#70) → Investigate #5
- Tahoe permissions (#51, #52) → Investigate #6
- App crashes (#76) → Investigate #1

Use this as a living document - update as you discover answers and add new questions as they arise during development.
