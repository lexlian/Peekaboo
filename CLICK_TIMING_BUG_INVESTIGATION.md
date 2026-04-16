# Click Command Timing Bug Investigation

## Issue Description
The `peekaboo click` command takes too long to execute and clicks through 2 application windows instead of just the target window.

## Root Cause Analysis

### Current Click Flow Timing

1. **Main Click** (AXorcist `Element.clickAt`):
   - Mouse down → 10ms delay → Mouse up
   - Total: ~10ms

2. **Post-Click Delay** (ClickCommand.swift:339):
   - Fixed 20ms delay after click
   - Total: +20ms

3. **Text Input Focus Nudging** (ClickService.swift:186-194):
   - **THIS IS THE PROBLEM**
   - If target is not already a focused text input, sends up to 4 additional "nudge" clicks
   - Each nudge: click + 60ms delay
   - **Maximum additional time: 240ms (4 × 60ms)**
   - **Maximum additional clicks: 4**

### Click Timing Breakdown (Worst Case)

```
T+0ms:    Main click at (x, y)
T+10ms:   Click completes
T+20ms:   Post-click delay ends
T+20ms:   Nudge check fails → send nudge click 1 at (x, y-29)
T+80ms:   Nudge check fails → send nudge click 2 at (x, y-24)
T+140ms:  Nudge check fails → send nudge click 3 at (x, y-34)
T+200ms:  Nudge check fails → send nudge click 4 at (x, y-20)
T+260ms:  All nudges exhausted, method returns
```

**Total execution time: ~260ms** (instead of ~30ms without nudging)

### Why It Clicks Through Windows

The `nudgeTextInputFocusIfNeeded` method is designed to handle SwiftUI text inputs that report incorrect frame offsets. However:

1. **Window switching**: Between nudge clicks, the application/window focus can change
2. **Multiple targets**: Each nudge click lands at a different Y coordinate, potentially on different UI elements or windows
3. **No window validation**: The method doesn't verify the frontmost window remains the same between clicks
4. **Blind retrying**: Even if the first nudge click succeeds, it continues checking and may send more clicks

## Problematic Code

### ClickService.swift:186-194

```swift
private func nudgeTextInputFocusIfNeeded(
    afterClickAt point: CGPoint,
    clickType: ClickType,
    expectedIdentifier: String?) async throws
{
    guard clickType == .single else { return }

    // If we're already focused on a text input, don't introduce extra clicks.
    if self.isFocusedTextInput(expectedIdentifier: normalizedExpectedIdentifier) {
        return
    }

    // SwiftUI can report text input frames with a stable vertical offset (commonly ~28-32px).
    // Retry a handful of small Y nudges to land inside the actual editable region.
    let nudges: [CGFloat] = [-29, -24, -34, -20]

    for dy in nudges {
        let candidate = CGPoint(x: point.x, y: point.y + dy)
        try await self.performClick(at: candidate, clickType: .single)  // ← Extra click!
        try await Task.sleep(nanoseconds: 60_000_000) // 60ms            // ← Delay!

        if self.isFocusedTextInput(expectedIdentifier: normalizedExpectedIdentifier) {
            return
        }
    }
}
```

**Issues:**
1. ❌ Sends clicks without confirming they're needed
2. ❌ No validation that target window is still frontmost
3. ❌ Aggressive retry logic (4 attempts)
4. ❌ Each retry clicks at different coordinates
5. ❌ Called after EVERY single click, not just text inputs

## Proposed Fixes

### Option 1: Disable Nudging by Default (Recommended)

Make text input nudging opt-in via a flag:

```swift
@Flag(help: "Enable aggressive text input focus nudging (for SwiftUI apps)")
var nudgeTextInput = false
```

**Pros:**
- Eliminates the timing issue for most users
- Preserves functionality for cases that need it
- Makes behavior explicit and controllable

**Cons:**
- Users with SwiftUI apps may need to discover and use the flag

### Option 2: Conservative Nudging

Reduce aggressiveness:
- Only nudge if target element is known to be a text input
- Reduce from 4 attempts to 2
- Reduce delay from 60ms to 30ms
- Add window stability check between clicks

**Pros:**
- Still helps with SwiftUI offset issues
- Less likely to cause window-switching problems

**Cons:**
- Still adds latency (60ms vs 240ms)
- More complex logic

### Option 3: Remove Nudging Entirely

Delete the `nudgeTextInputFocusIfNeeded` method and all calls to it.

**Pros:**
- Simplest solution
- Predictable timing
- No accidental multi-window clicks

**Cons:**
- SwiftUI text inputs with offset issues won't work
- Users would need to use coordinate clicks as workaround

### Option 4: Smart Nudging with Validation

Add comprehensive validation:
- Capture frontmost window PID before first nudge
- Verify window hasn't changed before each nudge
- Stop immediately if focus is achieved
- Add maximum total time budget (100ms)

**Pros:**
- Safest approach
- Prevents clicking wrong windows
- Still helps with SwiftUI issues

**Cons:**
- Most complex implementation
- Still adds some latency

## Recommended Solution

**Implement Option 1 + Option 2 combined:**

1. Add `--nudge-text-input` flag (default: false)
2. Only nudge if:
   - Flag is set, OR
   - Target element is explicitly identified as a text input field
3. Reduce to 2 attempts max, 30ms delay each
4. Add window PID validation between clicks

This provides:
- ✅ Predictable default behavior (no surprise extra clicks)
- ✅ Opt-in for aggressive nudging when needed
- ✅ Reduced latency even when nudging is enabled
- ✅ Safety against clicking wrong windows

## Files to Modify

1. `Apps/CLI/Sources/PeekabooCLI/Commands/Interaction/ClickCommand.swift`
   - Add `--nudge-text-input` flag
   - Pass flag to ClickService

2. `Core/PeekabooAutomationKit/Sources/PeekabooAutomationKit/Services/UI/ClickService.swift`
   - Make `nudgeTextInputFocusIfNeeded` conditional
   - Add window PID validation
   - Reduce nudge attempts and delays
   - OR remove the method entirely

## Testing Strategy

1. **Baseline timing test**: Measure click execution time without nudging
2. **SwiftUI text input test**: Verify nudging still works when enabled
3. **Multi-window test**: Ensure clicks don't land on wrong windows
4. **Regression test**: Verify existing click tests still pass

## Related Issues

- Issue #90: Coordinate click focus mismatch
- SwiftUI text input frame offset problems (known AX limitation)
- Electron app focus issues (Claude Desktop, VS Code)

---

*Investigation date: April 16, 2026*
*Affected commands: `peekaboo click`, `peekaboo agent` (when using click tool)*
