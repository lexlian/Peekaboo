# AI Provider Configuration Deep Dive

## Overview

Peekaboo supports multiple AI providers through a layered configuration system with **built-in providers** and **custom providers**. This document explains how `aiProviders.providers` is parsed and how custom providers integrate.

---

## Configuration Precedence

Provider configuration follows this hierarchy (highest to lowest priority):

1. **Environment Variable**: `PEEKABOO_AI_PROVIDERS`
2. **Config File**: `~/.peekaboo/config.json` → `aiProviders.providers`
3. **Default**: `"openai/gpt-5.1,anthropic/claude-sonnet-4.5"`

### Example Config

```json
{
  "aiProviders": {
    "providers": "anthropic/claude-sonnet-4.5,openai/gpt-5.1",
    "anthropicApiKey": "{env:ANTHROPIC_API_KEY}",
    "openaiApiKey": "{env:OPENAI_API_KEY}"
  }
}
```

---

## Provider String Format

### Syntax

```
provider/model[,provider2/model2,...]
```

- **Comma-separated** list (first available provider is used)
- **Format**: `provider/model`
- **Examples**:
  - `"openai/gpt-5.1"`
  - `"anthropic/claude-sonnet-4.5,openai/gpt-5.1"` (fallback chain)
  - `"ollama/llama3.3:latest"`
  - `"my-custom-provider/my-model"` (custom provider)

### Parsing Logic

**File**: `Tachikoma/Sources/Tachikoma/Providers/ProviderParser.swift`

```swift
public static func parse(_ providerString: String) -> ProviderConfig? {
    // Splits on first "/"
    let provider = String(before "/")
    let model = String(after "/")
    return ProviderConfig(provider: provider, model: model)
}

public static func parseList(_ providersString: String) -> [ProviderConfig] {
    // Splits on "," and parses each
    return providersString.split(separator: ",").compactMap { parse(String($0)) }
}
```

---

## Built-in Providers

These providers are natively supported:

| Provider | Models | Config Key |
|----------|--------|------------|
| `openai` | gpt-5.1, gpt-5-mini, gpt-4o, etc. | `openaiApiKey` |
| `anthropic` | claude-sonnet-4.5, claude-opus-4.5, etc. | `anthropicApiKey` |
| `google` / `gemini` | gemini-2.5-pro, gemini-2.5-flash | `geminiApiKey` |
| `grok` / `xai` | grok-4, grok-3 | `xaiApiKey` |
| `ollama` | llama3.3, llava, qwen2.5, etc. | (no key needed) |
| `lmstudio` | Any LM Studio model | (no key needed) |
| `mistral` | mistral-large, etc. | `mistralApiKey` |
| `groq` | llama3-groq, etc. | `groqApiKey` |

**File**: `Tachikoma/Sources/Tachikoma/Providers/ProviderFactory.swift:16-85`

---

## Custom Providers

### What Are Custom Providers?

Custom providers let you configure **OpenAI-compatible** or **Anthropic-compatible** APIs that aren't natively built-in (e.g., enterprise proxies, self-hosted models, vendor-specific endpoints).

### Configuration Structure

**File**: `Core/PeekabooCore/Sources/PeekabooAutomation/Configuration/Configuration.swift:208-299`

```json
{
  "customProviders": {
    "my-enterprise-llm": {
      "name": "Enterprise LLM",
      "description": "Internal company LLM proxy",
      "type": "openai",  // or "anthropic"
      "options": {
        "baseURL": "https://llm.internal.company.com/v1",
        "apiKey": "{env:ENTERPRISE_LLM_KEY}",
        "headers": {
          "X-Custom-Header": "value"
        },
        "timeout": 30,
        "retryAttempts": 3
      },
      "models": {
        "fast-model": {
          "name": "fast-v1.0",
          "maxTokens": 8192,
          "supportsTools": true,
          "supportsVision": false
        },
        "smart-model": {
          "name": "smart-v2.5",
          "maxTokens": 32768,
          "supportsTools": true,
          "supportsVision": true
        }
      },
      "enabled": true
    }
  }
}
```

### How Custom Providers Are Used

**Two separate systems work together:**

#### 1. Peekaboo's ConfigurationManager

**File**: `Core/PeekabooCore/Sources/PeekabooAutomation/Configuration/ConfigurationManager.swift:662-735`

Manages custom provider **configuration storage**:

```swift
// Add custom provider
try configManager.addCustomProvider(provider, id: "my-enterprise-llm")

// Get custom provider
let provider = configManager.getCustomProvider(id: "my-enterprise-llm")

// List all custom providers
let providers = configManager.listCustomProviders()

// Test custom provider connectivity
let (success, error) = await configManager.testCustomProvider(id: "my-enterprise-llm")
```

#### 2. Tachikoma's CustomProviderRegistry

**File**: `Tachikoma/Sources/Tachikoma/Core/CustomProviders.swift`

Loads custom providers **from config.json** and makes them available to `ProviderFactory`:

```swift
// Load from ~/.peekaboo/config.json
CustomProviderRegistry.shared.loadFromProfile()

// Get a custom provider
let info = CustomProviderRegistry.shared.get("my-enterprise-llm")

// Returns CustomProviderInfo with:
// - id: "my-enterprise-llm"
// - kind: .openai or .anthropic
// - baseURL: "https://..."
// - headers: [...]
// - models: ["fast-model": "fast-v1.0", ...]
```

**⚠️ Important**: The registry's `loadFromProfile()` must be called **before** creating providers. 

**✅ Fixed (2026-04-02)**: The registry is now automatically loaded during `PeekabooServices.refreshAgentService()` in `Core/PeekabooCore/Sources/PeekabooCore/Support/PeekabooServices.swift:403`.

---

## Custom Provider Flow

### Using Custom Provider in `aiProviders.providers`

```json
{
  "aiProviders": {
    "providers": "my-enterprise-llm/fast-model,openai/gpt-5.1"
  },
  "customProviders": {
    "my-enterprise-llm": { ... }
  }
}
```

### Resolution Process

1. **Parse providers string**: `"my-enterprise-llm/fast-model"` → `provider="my-enterprise-llm"`, `model="fast-model"`

2. **ProviderFactory.createProvider()** is called with `LanguageModel.custom(...)`

3. **Check CustomProviderRegistry** (`ProviderFactory.swift:86-107`):
   ```swift
   case let .custom(provider):
       if let parsed = ProviderParser.parse(provider.modelId) {
           if let custom = CustomProviderRegistry.shared.get(parsed.provider) {
               switch custom.kind {
               case .openai:
                   return try OpenAICompatibleProvider(
                       modelId: parsed.model,
                       baseURL: custom.baseURL,
                       configuration: configuration)
               case .anthropic:
                   return try AnthropicCompatibleProvider(
                       modelId: parsed.model,
                       baseURL: custom.baseURL,
                       configuration: configuration)
               }
           }
       }
       return provider
   ```

4. **Create compatible provider**: Uses `OpenAICompatibleProvider` or `AnthropicCompatibleProvider` with the custom `baseURL`

---

## Key Files

| File | Purpose |
|------|---------|
| `Core/PeekabooCore/Sources/PeekabooAutomation/Configuration/ConfigurationManager.swift` | Custom provider CRUD operations |
| `Core/PeekabooCore/Sources/PeekabooAutomation/Configuration/Configuration.swift` | CustomProvider, ProviderOptions, ModelDefinition structs |
| `Tachikoma/Sources/Tachikoma/Core/CustomProviders.swift` | CustomProviderRegistry that loads from config.json |
| `Tachikoma/Sources/Tachikoma/Providers/ProviderFactory.swift` | Creates providers, resolves custom providers via registry |
| `Tachikoma/Sources/Tachikoma/Providers/ProviderParser.swift` | Parses `provider/model` strings |
| `Tachikoma/Sources/Tachikoma/Providers/Compatible/OpenAICompatibleProvider.swift` | Handles custom OpenAI-compatible endpoints |
| `Tachikoma/Sources/Tachikoma/Providers/Compatible/AnthropicCompatibleProvider.swift` | Handles custom Anthropic-compatible endpoints |

---

## Current Gap: Registry Not Loaded at Startup

**Issue**: `CustomProviderRegistry.shared.loadFromProfile()` is only called in tests, not during normal Peekaboo startup.

**Impact**: Custom providers defined in `config.json` won't be available to `ProviderFactory` unless the registry is explicitly loaded.

**Solution**: Add registry initialization during `PeekabooServices` startup:

```swift
// In PeekabooServices.swift initialization
@MainActor
public func refreshAgentService() {
    // Load custom providers from config.json
    CustomProviderRegistry.shared.loadFromProfile()
    
    // ... rest of initialization
}
```

---

## Example Scenarios

### Scenario 1: Using Only Built-in Providers

```json
{
  "aiProviders": {
    "providers": "anthropic/claude-sonnet-4.5,openai/gpt-5.1",
    "anthropicApiKey": "sk-ant-...",
    "openaiApiKey": "sk-..."
  }
}
```

**Flow**: 
- Parser reads `"anthropic/claude-sonnet-4.5"`
- ProviderFactory creates `AnthropicProvider` directly
- No custom provider lookup needed

---

### Scenario 2: Using Custom Provider with Fallback

```json
{
  "aiProviders": {
    "providers": "enterprise-llm/fast-model,anthropic/claude-sonnet-4.5"
  },
  "customProviders": {
    "enterprise-llm": {
      "name": "Enterprise LLM",
      "type": "openai",
      "options": {
        "baseURL": "https://llm.internal/v1",
        "apiKey": "{env:ENTERPRISE_KEY}"
      },
      "models": {
        "fast-model": {
          "name": "fast-v1"
        }
      }
    }
  }
}
```

**Flow**:
1. Parser reads `"enterprise-llm/fast-model"`
2. ProviderFactory checks `CustomProviderRegistry.shared.get("enterprise-llm")`
3. ✅ Found! Creates `OpenAICompatibleProvider(baseURL: "https://llm.internal/v1")`
4. If custom provider fails, fallback to `"anthropic/claude-sonnet-4.5"`

---

### Scenario 3: Environment Variable Override

```bash
export PEEKABOO_AI_PROVIDERS="custom-provider/my-model"
peekaboo agent "Analyze this screen"
```

```json
{
  "customProviders": {
    "custom-provider": {
      "name": "My Custom",
      "type": "anthropic",
      "options": {
        "baseURL": "https://custom.api.com/v1",
        "apiKey": "{env:CUSTOM_API_KEY}"
      }
    }
  }
}
```

**Flow**:
1. Environment variable takes precedence over config file
2. Parser reads `"custom-provider/my-model"`
3. ProviderFactory resolves custom provider from registry
4. Creates `AnthropicCompatibleProvider(baseURL: "https://custom.api.com/v1")`

---

## CLI Commands

```bash
# View current provider configuration
peekaboo config show

# Add custom provider
peekaboo config providers add my-llm \
  --type openai \
  --base-url https://llm.internal/v1 \
  --api-key ENV_VAR_NAME \
  --models "fast=fast-v1,smart=smart-v2"

# List custom providers
peekaboo config providers list

# Test custom provider
peekaboo config providers test my-llm

# Remove custom provider
peekaboo config providers remove my-llm
```

---

## API Key Resolution

Custom provider API keys use **credential references**:

```json
"apiKey": "{env:ENTERPRISE_KEY}"  // References environment variable
```

**Resolution order** (`ConfigurationManager.swift:748-768`):
1. Environment variable
2. Credentials file (`~/.peekaboo/credentials.json`)
3. Config file value (if not a reference)

---

## Testing Custom Providers

```bash
# Test connectivity
peekaboo config providers test my-enterprise-llm

# Discover available models
peekaboo config providers discover my-enterprise-llm

# Use in agent mode
peekaboo agent --model "my-enterprise-llm/fast-model" "Summarize this window"
```

---

## Summary

| Question | Answer |
|----------|--------|
| **How is `aiProviders.providers` parsed?** | Comma-separated list of `provider/model` strings, parsed by `ProviderParser.parseList()` |
| **What happens with custom provider values?** | Parser extracts provider ID, `ProviderFactory` looks it up in `CustomProviderRegistry` |
| **How is `customProviders` section used?** | Stores configuration for non-standard providers; loaded by registry at startup (currently not called automatically) |
| **What if custom provider is not in registry?** | Falls back to next provider in the list, or uses default if none available |
| **Can I mix built-in and custom?** | Yes! Example: `"custom-llm/model,openai/gpt-5.1"` tries custom first, then OpenAI |
