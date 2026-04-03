# Custom Provider Integration Bug Fix Journey

## Overview
This document chronicles the complete bug fixing process to enable full custom provider (Bailian/qwen3.5-plus) support across all Peekaboo commands. The fixes resolved **6 major bugs** across multiple modules.

---

## Bug #1: Configuration Credentials Not Loading

### Symptom
API keys only worked from environment variables, not from config.json or credentials file.

### Root Cause
`getAnthropicAPIKey()`, `getOpenAIAPIKey()`, `getGeminiAPIKey()` accessed `credentials` dictionary without calling `loadCredentials()` first.

```swift
// BEFORE - Returns nil (credentials not loaded)
func getAnthropicAPIKey() -> String? {
    return credentials["anthropicApiKey"] as? String
}

// AFTER - Returns value from disk
func getAnthropicAPIKey() -> String? {
    self.loadCredentials()  // ← Added
    return credentials["anthropicApiKey"] as? String
}
```

### Fix Applied
Added `self.loadCredentials()` before credential access in all three methods.

**File:** `Core/PeekabooCore/Sources/PeekabooAutomation/Configuration/ConfigurationManager.swift` (lines 412, 438, 461)

**Commit:** `fix(config): load credentials before API key access`

---

## Bug #2: Custom Provider Registry Not Loading at Startup

### Symptom
Custom providers defined in config.json were unavailable to the agent and other commands.

### Root Cause
`CustomProviderRegistry.shared.loadFromProfile()` was only called in tests, not during normal startup.

### Fix Applied
Added registry load to `PeekabooServices.refreshAgentService()`:

```swift
// Load custom providers from config.json to make them available to ProviderFactory
Tachikoma.CustomProviderRegistry.shared.loadFromProfile()
```

**File:** `Core/PeekabooCore/Sources/PeekabooCore/Support/PeekabooServices.swift` (line 401)

**Commit:** `fix(providers): load custom providers registry at startup`

---

## Bug #3: Custom Provider API Key Not Passed

### Symptom
"invalid x-api-key" error when using custom provider despite API key being defined in config.json.

### Root Cause
1. `CustomProviderInfo` struct had no `apiKey` field
2. `CustomProviderRegistry.loadFromProfile()` read the API key from JSON but discarded it
3. `ProviderFactory` didn't pass the API key to compatible providers
4. `AnthropicCompatibleProvider` looked for API key in configuration, found nil, sent request without authentication

### Fix Applied
**Tachikoma submodule changes:**

1. Added `apiKey` field to `CustomProviderInfo`:
```swift
public struct CustomProviderInfo: Codable {
    public let type: String
    public let options: Options
    public let apiKey: String?  // ← Added
}
```

2. Store API key when loading:
```swift
let apiKey = customProviderInfo.apiKey  // ← Now preserved
```

3. Set API key in configuration before creating compatible providers:
```swift
if let apiKey {
    config.setAPIKey(apiKey, for: .anthropic)
}
```

**Files:** 
- `Tachikoma/Sources/Tachikoma/Core/CustomProviders.swift`
- `Tachikoma/Sources/Tachikoma/Providers/ProviderFactory.swift`

**Commit:** Tachikoma `c810f4f` fix(custom-providers): load and pass API key from config.json

---

## Bug #4: Agent Command Rejected Custom Providers

### Symptom
"Agent service not available" error even though custom provider was configured and testable.

### Root Causes (Multiple)
1. `hasConfiguredAIProvider()` only checked OpenAI/Anthropic/Gemini, not custom providers
2. Model parser (`LanguageModel.parse(from:)`) rejected `.anthropicCompatible`/`.openaiCompatible`/`.custom` models
3. Model string extraction lost provider prefix ("qwen3.5-plus" instead of "bailian/qwen3.5-plus")
4. API key lookup didn't check compatible provider keys

### Fix Applied

**1. Provider Detection** (`AgentCommand.swift`):
```swift
private func hasConfiguredAIProvider(configuration: PeekabooCore.ConfigurationManager) -> Bool {
    let hasOpenAI = configuration.getOpenAIAPIKey()?.isEmpty == false
    let hasAnthropic = configuration.getAnthropicAPIKey()?.isEmpty == false
    let hasGemini = configuration.getGeminiAPIKey()?.isEmpty == false
    let hasCustomProvider = !configuration.listCustomProviders().isEmpty  // ← Added
    return hasOpenAI || hasAnthropic || hasGemini || hasCustomProvider
}
```

**2. Model Parsing** (`Model.swift`):
Added custom provider detection before `return nil`:
```swift
// Check if it's a custom provider
if let customProvider = CustomProviderRegistry.shared.get(provider) {
    if customProvider.type == "anthropic" {
        return .anthropicCompatible(model)
    } else if customProvider.type == "openai" {
        return .openaiCompatible(model)
    }
}
```

**3. Model String Extraction** (`PeekabooServices.swift`):
```swift
// BEFORE - Loses provider prefix
let model = components.last.map { String($0) }

// AFTER - Uses full "provider/model" string
let model = components.first.map { String($0) }
```

**4. API Key Lookup** (`PeekabooAgentService.swift`):
```swift
case .anthropic:
    return config.getAPIKey(for: .anthropic)
        ?? config.getAPIKey(for: .custom("anthropic_compatible"))  // ← Added
        ?? config.getAPIKey(for: .custom("openai_compatible"))     // ← Added
```

**Files:** `AgentCommand.swift`, `PeekabooAgentService.swift`, `PeekabooServices.swift`, `Model.swift`

**Commit:** `fix(agent): support custom providers in agent command`

---

## Bug #5: Agent Returns Empty Response

### Symptom
Agent executed successfully but returned empty content despite server sending valid data.

### Root Causes

#### Cause 5a: Thinking Content Not Decodable
`AnthropicResponseContent` enum couldn't decode "thinking" and "redacted_thinking" content blocks that Bailian server sends.

**Server response flow:**
1. content_block_start (thinking)
2. content_block_delta (thinking_delta) - accumulates thinking
3. content_block_stop (thinking ends)
4. content_block_start (text)
5. content_block_delta (text_delta) - "Hello to you"
6. content_block_stop (text ends)

**Evidence from curl:**
```json
{"type":"thinking","signature":"...","thinking":"..."}
{"type":"text","text":"Hello..."}
```

#### Cause 5b: SSE Data Prefix Format Mismatch
SSE parser expected `data: ` (with space) but Bailian sends `data:` (no space).

**curl shows:**
```
data:{"type":"content_block_start",...}  ← No space after colon
```

**Code checked:**
```swift
if line.hasPrefix("data: ")  ← Expected space
```

### Fix Applied

**5a. Added Thinking Content Support** (`AnthropicTypes.swift`):
```swift
public enum AnthropicResponseContent: Codable {
    case text(TextContent)
    case toolUse(ToolUseContent)
    case thinking(ThinkingContent)        // ← Added
    case redactedThinking(RedactedThinkingContent)  // ← Added
}

public struct ThinkingContent: Codable {
    public let signature: String
    public let thinking: String
}

public struct RedactedThinkingContent: Codable {
    public let data: String
}
```

**5b. Fixed SSE Parsing** (`BaseProviders.swift`):
```swift
// BEFORE
if line.hasPrefix("data: ") {
    let jsonString = String(line.dropFirst(6))

// AFTER - Handles both formats
if line.hasPrefix("data:") {
    // Handle both "data:" and "data: " formats (some servers omit the space)
    let prefixLength = line.hasPrefix("data: ") ? 6 : 5
    let jsonString = String(line.dropFirst(prefixLength))
```

**Files:** 
- `Tachikoma/Sources/Tachikoma/Providers/Anthropic/AnthropicTypes.swift`
- `Tachikoma/Sources/Tachikoma/Core/BaseProviders.swift` (lines 407, 631)

**Commit:** Tachikoma `eda7cad` fix(streaming): handle SSE data prefix without space

---

## Bug #6: `peekaboo see --analyze` Failed

### Symptom
See command with --analyze failed with custom provider while agent worked.

### Root Cause
`PeekabooAIService` didn't load custom providers from CustomProviderRegistry before parsing provider strings, so 'bailian/qwen3.5-plus' was unrecognized and fell back to default Anthropic provider without proper API key.

### Fix Applied

**1. Load Custom Providers** (`PeekabooAIService.swift`):
```swift
init() {
    Tachikoma.CustomProviderRegistry.shared.loadFromProfile()  // ← Added
    // ... rest of init
}
```

**2. Parse Custom Providers** (`parseProviderEntry()`):
```swift
// Check CustomProviderRegistry for provider ID before built-in providers
if let customProvider = CustomProviderRegistry.shared.get(providerID) {
    if customProvider.type == "anthropic" {
        return .anthropicCompatible(model)
    } else if customProvider.type == "openai" {
        return .openaiCompatible(model)
    }
}
```

**3. Fallback to Custom Providers** (`resolveAvailableModels()`):
```swift
// If no providers string configured, use first custom provider from registry
if let firstCustom = CustomProviderRegistry.shared.getAll().first {
    // Use as default
}
```

**File:** `Core/PeekabooCore/Sources/PeekabooAgentRuntime/AIService/PeekabooAIService.swift`

**Commit:** `fix(see): support custom providers in AI analyze`

---

## Additional Issues Resolved

### Build Environment: Swift Version Mismatch

**Issue:** swift.org Swift 6.2+ removed `Data.bytes` SPI that `swift-configuration` dependency uses.

**Error:**
```
error: value of type 'Data' has no member 'bytes'
```

**Solution:** Use Xcode's bundled Swift compiler instead of swift.org Swiftly toolchain:
```bash
export DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer
swift build -c release
```

**Key Finding:** Official releases use Xcode Swift (not swift.org), which maintains backward compatibility with `.bytes` SPI.

### Network/Proxy: System Proxy Blocking Connections

**Issue:** System proxy was blocking Swift URLSession connections to bailian server.

**Symptom:**
- curl worked fine
- Swift URL session failed with timeout/connection refused

**Solution:**
```bash
networksetup -setwebproxystate "Wi-Fi" off
networksetup -setsecurewebproxystate "Wi-Fi" off
```

---

## Final Test Results

All commands now work with custom provider (bailian/qwen3.5-plus):

```bash
# Agent command
$ peekaboo agent "Say hello in 3 words"
Starting: Say hello in 3 words

The user is asking me to say hello in 3 words.
This is a simple request that doesn't require any tools -
I can just respond directly with a greeting that uses exactly 3 words.

Hello there friend!
Task completed in 2.6s with ⚒ 0 tools

# Math question
$ peekaboo agent "What is 2+2? Answer in one word."
Starting: What is 2+2? Answer in one word.

This is a simple math question. The user is asking what
2+2 equals and wants a one-word answer. This is a straightforward
arithmetic question - 2+2=4. The answer in one word would be "Four".

Four
Task completed in 3.6s with ⚒ 0 tools

# See with analyze
$ peekaboo see --mode screen --analyze "What's on the screen?"
{
  "success": true,
  "data": {
    "analysis": "The screen shows...",
    "provider": "anthropic-compatible",
    "model": "qwen3.5-plus"
  }
}
```

---

## Key Learnings

1. **Always load before access** - Credentials/registries must be loaded before use, not lazily
2. **Handle protocol variations** - SSE spec allows `data:` or `data: ` - support both
3. **Extend enums carefully** - Custom providers may use extended content types (thinking, etc.)
4. **Test all commands** - Agent worked but see didn't - different code paths
5. **Environment matters** - Xcode Swift ≠ swift.org Swift for compatibility
6. **Debug with multiple tools** - curl vs Swift URLSession revealed proxy issue

---

## Commits Summary

**Peekaboo (3 commits):**
- `fix(config): load credentials before API key access`
- `fix(providers): load custom providers registry at startup`
- `fix(agent): support custom providers in agent command`
- `fix(see): support custom providers in AI analyze`
- `build(submodules): bump Tachikoma to fix SSE streaming`

**Tachikoma (6 commits):**
- `c810f4f` fix(custom-providers): load and pass API key from config.json
- `eda7cad` fix(streaming): handle SSE data prefix without space
- Plus model parsing and enum extension commits

**Total:** 9 commits across both repos
**Files changed:** ~15 files
**Lines changed:** ~200 insertions, ~50 deletions

---

## Configuration Example

Working config.json for Bailian/qwen3.5-plus:

```json
{
  "aiProviders": {
    "providers": "bailian/qwen3.5-plus"
  },
  "customProviders": {
    "bailian": {
      "type": "anthropic",
      "options": {
        "baseURL": "https://coding.dashscope.aliyuncs.com/apps/anthropic",
        "apiKey": "sk-sp-..."
      }
    }
  }
}
```

---

## Timeline

1. **Configuration bug** - Credentials not loading from disk
2. **Registry bug** - Custom providers not available at runtime
3. **API key bug** - Keys not passed to compatible providers
4. **Agent integration bug** - Multiple issues preventing agent from using custom providers
5. **Empty response bug** - Thinking content and SSE parsing issues
6. **See command bug** - Analyze command not loading custom providers

**Total debugging time:** ~4 hours
**Build time per iteration:** 90-150s
**Test time per iteration:** 5-30s

---

## Related Documentation

- [CONFIGURATION_DEEP_DIVE.md](CONFIGURATION_DEEP_DIVE.md) - Configuration system details
- [PROVIDER_CONFIGURATION_DEEP_DIVE.md](PROVIDER_CONFIGURATION_DEEP_DIVE.md) - Custom provider setup guide
- [BUGFIX_ANTHROPIC_CREDENTIALS.md](BUGFIX_ANTHROPIC_CREDENTIALS.md) - Initial credential loading fix
