# Configuration Management Deep Dive

*Last updated: 2026-03-31*  
*Module: Core/PeekabooCore/Sources/PeekabooAutomation/Configuration/*

---

## Overview

Peekaboo's configuration management system implements a **hierarchical precedence model** with support for:
- JSONC format (JSON with Comments)
- Environment variable expansion (`${VAR_NAME}`)
- Separate credentials storage with file permissions
- Custom AI provider definitions
- Runtime configuration editing and validation

---

## Module Structure

```
Core/PeekabooCore/Sources/PeekabooAutomation/Configuration/
├── Configuration.swift          # Data models (300 lines)
└── ConfigurationManager.swift   # Management logic (1066 lines)

Apps/CLI/Sources/PeekabooCLI/
├── CLI/Configuration/
│   └── CLIConfiguration.swift   # CLI-specific wrapper (170 lines)
└── Commands/Core/
    ├── ConfigCommand.swift                 # Main command entry (51 lines)
    ├── ConfigCommand+InitShowEdit.swift    # init/show/edit subcommands (305 lines)
    ├── ConfigCommand+Providers.swift       # provider management (693 lines)
    ├── ConfigCommand+AddLogin.swift        # add/login subcommands
    ├── ConfigCommand+ValidateCredential.swift  # validation logic
    ├── ConfigCommand+Status.swift          # status display
    ├── ConfigCommand+Bindings.swift        # key bindings
    └── ConfigCommand+Shared.swift          # shared utilities
```

---

## Configuration Precedence

**Highest to lowest priority:**

1. **Command-line arguments** - e.g., `peekaboo see --providers "openai/gpt-5.1"`
2. **Environment variables** - e.g., `PEEKABOO_AI_PROVIDERS=...`
3. **Credentials file** (`~/.peekaboo/credentials`) - OAuth tokens, API keys
4. **Configuration file** (`~/.peekaboo/config.json`) - JSONC format
5. **Built-in defaults** - hardcoded fallbacks

**Example: AI Provider Selection**
```swift
// User runs: peekaboo agent "..."
// System resolves model through:
1. CLI flag: --model "anthropic/claude-sonnet-4.5"        ← if provided
2. Env var:  PEEKABOO_AI_PROVIDERS                        ← else check
3. Config:   config.json → aiProviders.providers          ← else check
4. Default:  "openai/gpt-5.1,anthropic/claude-sonnet-4.5" ← fallback
```

---

## File Locations

### Primary Configuration

| File | Path | Permissions | Description |
|------|------|-------------|-------------|
| Config | `~/.peekaboo/config.json` | 0700 (dir) | JSONC format, all settings |
| Credentials | `~/.peekaboo/credentials` | 0600 | KEY=value pairs, sensitive data |
| Logs | `~/.peekaboo/logs/peekaboo.log` | - | Runtime logs |

**Override via environment:**
```bash
export PEEKABOO_CONFIG_DIR=/custom/path  # Changes base directory
```

### Legacy Paths (Auto-migrated)

- Old config: `~/.config/peekaboo/config.json`
- Migration happens automatically on first run
- API keys extracted from old config → moved to credentials file

---

## Configuration Schema

### Root Structure (Configuration.swift)

```swift
public struct Configuration: Codable {
    public var aiProviders: AIProviderConfig?      // AI models & keys
    public var defaults: DefaultsConfig?           // Capture defaults
    public var logging: LoggingConfig?             // Log settings
    public var agent: AgentConfig?                 // Agent behavior
    public var visualizer: VisualizerConfig?       // Visual feedback
    public var tools: ToolConfig?                  // Tool allow/deny lists
    public var customProviders: [String: CustomProvider]?  // Custom AI endpoints
}
```

### AI Provider Configuration

```swift
// config.json
{
  "aiProviders": {
    "providers": "openai/gpt-5.1,anthropic/claude-sonnet-4.5",
    "openaiApiKey": "${OPENAI_API_KEY}",  // Env var reference
    "anthropicApiKey": "${ANTHROPIC_API_KEY}",
    "ollamaBaseUrl": "http://localhost:11434"
  }
}
```

**Fields:**
- `providers` - Comma-separated list of `provider/model` pairs
- `openaiApiKey` - OpenAI key (or reference)
- `anthropicApiKey` - Anthropic key (or reference)
- `ollamaBaseUrl` - Local Ollama endpoint

### Defaults Configuration

```swift
{
  "defaults": {
    "savePath": "~/Desktop/Screenshots",
    "imageFormat": "png",
    "captureMode": "window",
    "captureFocus": "auto"
  }
}
```

### Agent Configuration

```swift
{
  "agent": {
    "defaultModel": "anthropic/claude-sonnet-4.5",
    "maxSteps": 50,
    "showThoughts": true,
    "temperature": 0.7,
    "maxTokens": 16384
  }
}
```

### Visualizer Configuration

```swift
{
  "visualizer": {
    "enabled": true,
    "animationSpeed": 1.0,
    "effectIntensity": 0.8,
    "soundEnabled": false,
    "keyboardTheme": "modern",
    "clickAnimationEnabled": true,
    "typeAnimationEnabled": true,
    "screenshotFlashEnabled": true,
    // ... many more toggles
  }
}
```

### Custom Provider Example

```swift
{
  "customProviders": {
    "openrouter": {
      "name": "OpenRouter",
      "description": "Access to 300+ models",
      "type": "openai",
      "options": {
        "baseURL": "https://openrouter.ai/api/v1",
        "apiKey": "{env:OPENROUTER_API_KEY}",
        "headers": {
          "HTTP-Referer": "https://myapp.com",
          "X-Title": "My Peekaboo Instance"
        },
        "timeout": 30.0,
        "retryAttempts": 3
      },
      "models": {
        "meta-llama/llama-3-70b-instruct": {
          "name": "Llama 3 70B",
          "maxTokens": 8192,
          "supportsTools": true,
          "supportsVision": false
        }
      },
      "enabled": true
    }
  }
}
```

---

## Key Components

### ConfigurationManager (Core Manager)

**Singleton pattern:**
```swift
public static let shared = ConfigurationManager()
```

**Core responsibilities:**
1. Load/parse JSONC configuration
2. Resolve environment variable expansion
3. Manage credentials file (read/write with permissions)
4. Provide precedence-based value resolution
5. Custom provider CRUD operations
6. Provider connectivity testing

**Key methods:**

```swift
// Load configuration
public func loadConfiguration() -> Configuration?

// Get value with precedence resolution
public func getValue<T>(
    cliValue: T?,
    envVar: String?,
    configValue: T?,
    defaultValue: T) -> T

// Credential management
public func saveCredentials(_ newCredentials: [String: String]) throws
public func setCredential(key: String, value: String) throws
public func credentialValue(for key: String) -> String?

// Custom providers
public func addCustomProvider(_ provider: Configuration.CustomProvider, id: String) throws
public func testCustomProvider(id: String) async -> (success: Bool, error: String?)
public func discoverModelsForCustomProvider(id: String) async -> (models: [String], error: String?)
```

### JSONC Parser

**Handles:**
- Single-line comments: `// comment`
- Multi-line comments: `/* comment */`
- Environment variable expansion: `${VAR_NAME}`
- Tilde expansion: `~/Desktop` → `/Users/username/Desktop`

**Implementation:**
```swift
public func stripJSONComments(from json: String) -> String {
    var stripper = JSONCommentStripper(json: json)
    return stripper.strip()
}

public func expandEnvironmentVariables(in text: String) -> String {
    let pattern = #"\$\{([A-Za-z_][A-Za-z0-9_]*)\}"#
    // ... regex replacement
}
```

### JSONCommentStripper

**State machine parser:**
```swift
private struct JSONCommentStripper {
    private let characters: [Character]
    private var index: Int = 0
    private var result = ""
    private var inString = false
    private var escapeNext = false
    private var singleLineComment = false
    private var multiLineComment = false
    
    mutating func strip() -> String {
        while index < characters.count {
            let char = characters[index]
            let next = peek()
            
            if handleEscape(char) { continue }
            if handleQuote(char) { continue }
            if inString { append(char); advance(); continue }
            if handleCommentStart(char, next) { continue }
            if handleCommentEnd(char, next) { continue }
            appendIfNeeded(char)
            advance()
        }
        return result
    }
}
```

**State transitions:**
- `inString` → inside quoted string (preserve content)
- `escapeNext` → backslash in string (escape next char)
- `singleLineComment` → after `//` (skip until newline)
- `multiLineComment` → after `/*` (skip until `*/`)

---

## CLI Commands

### `peekaboo config init`

**Creates default configuration:**
```bash
peekaboo config init [--force] [--timeout-seconds 30]
```

**Implementation:**
```swift
struct InitCommand: ConfigRuntimeCommand {
    @Flag var force = false
    @Option var timeoutSeconds: Double = 30
    
    mutating func run(using runtime: CommandRuntime) async throws {
        try ensureWritableConfig(at: path)  // Check exists
        try createConfiguration(at: path)    // Write template
        let reporter = ProviderStatusReporter()
        await reporter.printSummary()        // Test providers
    }
}
```

**Default template:**
```json
{
  "aiProviders": {
    "providers": "openai/gpt-5.1,anthropic/claude-sonnet-4.5"
  },
  "defaults": {
    "savePath": "~/Desktop/Screenshots",
    "imageFormat": "png",
    "captureMode": "window",
    "captureFocus": "auto"
  },
  "logging": {
    "level": "info",
    "path": "~/.peekaboo/logs/peekaboo.log"
  }
}
```

### `peekaboo config show`

**Display configuration:**
```bash
peekaboo config show [--effective] [--json]
```

**Modes:**
- Default → shows raw config.json content
- `--effective` → shows merged values (after precedence resolution)
- `--json` → JSON output format

**Effective config example:**
```
Effective Configuration (after merging all sources):
==================================================

AI Providers:
  Providers: openai/gpt-5.1,anthropic/claude-sonnet-4.5
  OpenAI API Key: ***SET***
  Ollama Base URL: http://localhost:11434

Defaults:
  Save Path: /Users/lex/Desktop/Screenshots

Logging:
  Level: info
  Path: /Users/lex/.peekaboo/logs/peekaboo.log

Files:
  Config File: /Users/lex/.peekaboo/config.json
  Credentials: /Users/lex/.peekaboo/credentials
```

### `peekaboo config edit`

**Open in editor:**
```bash
peekaboo config edit [--editor code] [--print-path]
```

**Implementation:**
```swift
struct EditCommand: ConfigRuntimeCommand {
    @Option var editor: String?
    @Flag var printPath: Bool
    
    mutating func run(using runtime: CommandRuntime) async throws {
        if printPath {
            print(self.configPath)
            return
        }
        
        // Create if doesn't exist
        if !FileManager.default.fileExists(atPath: self.configPath) {
            try self.configManager.createDefaultConfiguration()
        }
        
        // Open editor
        let editorCommand = self.editor ?? self.defaultEditor()
        let process = Process()
        process.executableURL = URL(fileURLWithPath: "/usr/bin/env")
        process.arguments = [editorCommand, self.configPath]
        try process.run()
        process.waitUntilExit()
        
        // Validate
        if self.configManager.loadConfiguration() != nil {
            print("[ok] Configuration is valid.")
        } else {
            print("[warn] Configuration may be invalid.")
        }
    }
}
```

### `peekaboo config add-provider`

**Add custom AI provider:**
```bash
peekaboo config add-provider openrouter \
  --type openai \
  --name "OpenRouter" \
  --base-url "https://openrouter.ai/api/v1" \
  --api-key "{env:OPENROUTER_API_KEY}" \
  --description "Access to 300+ models" \
  [--dry-run] [--force] [--headers "Key:Value,Key2:Value2"]
```

**Validation:**
1. Provider ID format (letters, numbers, hyphens)
2. Provider type (`openai` or `anthropic`)
3. Name not empty
4. URL has scheme and host
5. API key not empty
6. Headers format (if provided)

**Dry run:**
```bash
peekaboo config add-provider myprovider --dry-run
# Shows what would be written without saving
```

### `peekaboo config test-provider`

**Test connectivity:**
```bash
peekaboo config test-provider openrouter
```

**Implementation:**
```swift
public func testCustomProvider(id: String) async -> (success: Bool, error: String?) {
    guard let provider = getCustomProvider(id: id) else {
        return (false, "Provider '\(id)' not found")
    }
    
    guard let apiKey = resolveCredential(provider.options.apiKey) else {
        return (false, "API key not found")
    }
    
    switch provider.type {
    case .openai:
        return try await testOpenAICompatibleProvider(provider: provider, apiKey: apiKey)
    case .anthropic:
        return try await testAnthropicCompatibleProvider(provider: provider, apiKey: apiKey)
    }
}

private func testOpenAICompatibleProvider(...) async throws -> (success: Bool, error: String?) {
    let url = URL(string: "\(provider.options.baseURL)/models")!
    var request = URLRequest(url: url)
    request.httpMethod = "GET"
    request.setValue("Bearer \(apiKey)", forHTTPHeaderField: "Authorization")
    
    let (data, response) = try await URLSession.shared.data(for: request)
    guard let httpResponse = response as? HTTPURLResponse else {
        return (false, "Invalid response")
    }
    
    guard httpResponse.statusCode == 200 else {
        let errorMessage = String(data: data, encoding: .utf8) ?? "HTTP \(httpResponse.statusCode)"
        return (false, errorMessage)
    }
    
    return (true, nil)
}
```

### `peekaboo config models-provider`

**Discover available models:**
```bash
peekaboo config models-provider openrouter
```

**Implementation:**
```swift
public func discoverModelsForCustomProvider(id: String) async -> (models: [String], error: String?) {
    guard let provider = getCustomProvider(id: id) else {
        return ([], "Provider '\(id)' not found")
    }
    
    guard let apiKey = resolveCredential(provider.options.apiKey) else {
        return ([], "API key not found")
    }
    
    switch provider.type {
    case .openai:
        return try await discoverOpenAICompatibleModels(provider: provider, apiKey: apiKey)
    case .anthropic:
        // Anthropic doesn't have models endpoint, return configured models
        let configuredModels = provider.models?.keys.map { String($0) } ?? []
        return (configuredModels, nil)
    }
}

private func discoverOpenAICompatibleModels(...) async throws -> (models: [String], error: String?) {
    let url = URL(string: "\(provider.options.baseURL)/models")!
    // ... GET request similar to test method
    
    struct ModelsResponse: Codable {
        let data: [ModelInfo]
        struct ModelInfo: Codable { let id: String }
    }
    
    let response = try JSONDecoder().decode(ModelsResponse.self, from: data)
    return (response.data.map(\.id), nil)
}
```

---

## Credential Management

### Credentials File Format

**Location:** `~/.peekaboo/credentials`  
**Permissions:** `0600` (owner read/write only)

```ini
# Peekaboo credentials file
# This file contains sensitive API keys and should not be shared
#
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GEMINI_API_KEY=...
OPENROUTER_API_KEY=...
```

### OAuth Token Support

**Access tokens with expiry:**
```ini
OPENAI_ACCESS_TOKEN=...
OPENAI_ACCESS_EXPIRES=1234567890  # Unix timestamp
ANTHROPIC_ACCESS_TOKEN=...
```

**Validation:**
```swift
private func validOAuthAccessToken(prefix: String) -> String? {
    self.loadCredentials()
    guard let token = self.credentials["\(prefix)_ACCESS_TOKEN"] else { return nil }
    guard let expiryString = self.credentials["\(prefix)_ACCESS_EXPIRES"],
          let expiryInt = Int(expiryString) else { return token }
    let expiryDate = Date(timeIntervalSince1970: TimeInterval(expiryInt))
    if expiryDate > Date() {
        return token  // Still valid
    }
    return nil  // Expired
}
```

### API Key Resolution Precedence

**For OpenAI:**
```swift
public func getOpenAIAPIKey() -> String? {
    // 1. Environment variable (highest priority)
    if let envValue = self.environmentValue(for: "OPENAI_API_KEY") {
        return envValue
    }
    
    // 2. OAuth access token (credentials)
    if let token = self.validOAuthAccessToken(prefix: "OPENAI") {
        return token
    }
    
    // 3. Credentials file
    if let credValue = credentials["OPENAI_API_KEY"] {
        return credValue
    }
    
    // 4. Config file (discouraged, but supported)
    if let configValue = configuration?.aiProviders?.openaiApiKey {
        return configValue
    }
    
    return nil
}
```

---

## Custom Provider Architecture

### Provider Types

```swift
public enum ProviderType: String, Codable, CaseIterable {
    case openai     // OpenAI-compatible API
    case anthropic  // Anthropic-compatible API
}
```

**Capabilities:**
- **OpenAI type:** GET `/models`, POST `/chat/completions` or `/responses`
- **Anthropic type:** POST `/messages`

### Provider Options

```swift
public struct ProviderOptions: Codable {
    public let baseURL: String
    public let apiKey: String  // Can be {env:VAR_NAME} reference
    public let headers: [String: String]?
    public let timeout: TimeInterval?
    public let retryAttempts: Int?
    public let defaultParameters: [String: String]?
}
```

### Model Definitions

```swift
public struct ModelDefinition: Codable {
    public let name: String
    public let maxTokens: Int?
    public let supportsTools: Bool?
    public let supportsVision: Bool?
    public let parameters: [String: String]?
}
```

**Usage:**
```swift
let provider = Configuration.CustomProvider(
    name: "My Provider",
    type: .openai,
    options: ProviderOptions(
        baseURL: "https://api.example.com/v1",
        apiKey: "{env:MY_API_KEY}",
        headers: ["X-Custom": "value"],
        timeout: 30.0
    ),
    models: [
        "fast-model": ModelDefinition(
            name: "Fast Model",
            maxTokens: 4096,
            supportsTools: false,
            supportsVision: false
        ),
        "smart-model": ModelDefinition(
            name: "Smart Model",
            maxTokens: 16384,
            supportsTools: true,
            supportsVision: true
        )
    ],
    enabled: true
)
```

---

## Value Resolution API

### Generic Value Resolution

```swift
public func getValue<T>(
    cliValue: T?,
    envVar: String?,
    configValue: T?,
    defaultValue: T) -> T
{
    // CLI argument takes highest precedence
    if let cliValue {
        return cliValue
    }
    
    // Environment variable takes second precedence
    if let envVar,
       let envValue = self.environmentValue(for: envVar),
       let converted: T = self.convertEnvValue(envValue, as: T.self)
    {
        return converted
    }
    
    // Config file value takes third precedence
    if let configValue {
        return configValue
    }
    
    // Default value as fallback
    return defaultValue
}

private func convertEnvValue<T>(_ value: String, as type: T.Type) -> T? {
    switch type {
    case is String.Type:
        return value as? T
    case is Bool.Type:
        return (value.lowercased() == "true" || value == "1") as? T
    case is Int.Type:
        return Int(value) as? T
    case is Double.Type:
        return Double(value) as? T
    default:
        return nil
    }
}
```

### Specific Getters

```swift
// AI providers
public func getAIProviders(cliValue: String? = nil) -> String {
    self.getValue(
        cliValue: cliValue,
        envVar: "PEEKABOO_AI_PROVIDERS",
        configValue: self.configuration?.aiProviders?.providers,
        defaultValue: "openai/gpt-5.1,anthropic/claude-sonnet-4.5")
}

// Ollama base URL
public func getOllamaBaseURL() -> String {
    self.getValue(
        cliValue: nil as String?,
        envVar: "PEEKABOO_OLLAMA_BASE_URL",
        configValue: self.configuration?.aiProviders?.ollamaBaseUrl,
        defaultValue: "http://localhost:11434")
}

// Default save path
public func getDefaultSavePath(cliValue: String? = nil) -> String {
    let path = self.getValue(
        cliValue: cliValue,
        envVar: "PEEKABOO_DEFAULT_SAVE_PATH",
        configValue: self.configuration?.defaults?.savePath,
        defaultValue: "~/Desktop")
    return NSString(string: path).expandingTildeInPath
}

// Log level
public func getLogLevel() -> String {
    self.getValue(
        cliValue: nil as String?,
        envVar: "PEEKABOO_LOG_LEVEL",
        configValue: self.configuration?.logging?.level,
        defaultValue: "info")
}
```

---

## Testing

### Test Files

```
Core/PeekabooCore/Tests/PeekabooTests/Configuration/
├── ToolConfigTests.swift         # Tool allow/deny lists
└── ConfigurationEnvironmentTests.swift  # Environment variable resolution
```

### Example Test

```swift
import Foundation
import PeekabooAutomation
import Testing

struct ToolConfigTests {
    @Test
    func `Codable round-trip for tool lists`() throws {
        let tools = Configuration.ToolConfig(allow: ["see", "click"], deny: ["shell"])
        let config = Configuration(tools: tools)
        
        let data = try JSONEncoder().encode(config)
        let decoded = try JSONDecoder().decode(Configuration.self, from: data)
        
        #expect(decoded.tools?.allow == ["see", "click"])
        #expect(decoded.tools?.deny == ["shell"])
    }
}
```

### CLI Tests

```
Apps/CLI/Tests/CLIAutomationTests/
├── ConfigCommandTests.swift         # CLI command behavior
├── ConfigGuidanceSnapshotTests.swift  # Config output snapshots
└── ConfigurationTests.swift         # Integration tests
```

---

## Environment Variables Reference

### Configuration Override

| Variable | Description | Example |
|----------|-------------|---------|
| `PEEKABOO_CONFIG_DIR` | Override config directory | `/etc/peekaboo` |
| `PEEKABOO_CONFIG_DISABLE_MIGRATION` | Skip legacy migration | `1` |
| `PEEKABOO_AI_PROVIDERS` | AI provider list | `openai/gpt-5.1,anthropic/...` |
| `PEEKABOO_OLLAMA_BASE_URL` | Ollama endpoint | `http://localhost:11434` |
| `PEEKABOO_DEFAULT_SAVE_PATH` | Default screenshot path | `~/Pictures` |
| `PEEKABOO_LOG_LEVEL` | Logging verbosity | `debug`, `info`, `warn` |
| `PEEKABOO_LOG_PATH` | Log file location | `/var/log/peekaboo.log` |
| `PEEKABOO_INCLUDE_AUTOMATION_TESTS` | Enable automation tests | `true` |
| `PEEKABOO_USE_MODERN_CAPTURE` | Use ScreenCaptureKit | `1` |
| `PEEKABOO_MENUBAR_OCR_VERIFY` | Menu bar OCR verification | `0` |

### API Keys (Alternative to credentials file)

| Variable | Provider | Description |
|----------|----------|-------------|
| `OPENAI_API_KEY` | OpenAI | API key or bearer token |
| `ANTHROPIC_API_KEY` | Anthropic | API key |
| `GEMINI_API_KEY` | Google | Gemini API key |
| `GROQ_API_KEY` | Groq | Groq API key |
| `OPENROUTER_API_KEY` | OpenRouter | OpenRouter API key |

### OAuth Tokens

| Variable | Description |
|----------|-------------|
| `OPENAI_ACCESS_TOKEN` | OpenAI OAuth token |
| `OPENAI_ACCESS_EXPIRES` | Token expiry (Unix timestamp) |
| `ANTHROPIC_ACCESS_TOKEN` | Anthropic OAuth token |

---

## Common Operations

### Initialize Configuration

```bash
# Create default config
peekaboo config init

# Force overwrite existing
peekaboo config init --force
```

### View Configuration

```bash
# Show raw config.json
peekaboo config show

# Show effective config (with env vars merged)
peekaboo config show --effective

# JSON output
peekaboo config show --json
```

### Edit Configuration

```bash
# Open in $EDITOR
peekaboo config edit

# Open in specific editor
peekaboo config edit --editor code

# Just print path
peekaboo config edit --print-path
```

### Manage Custom Providers

```bash
# Add provider
peekaboo config add-provider openrouter \
  --type openai \
  --name "OpenRouter" \
  --base-url "https://openrouter.ai/api/v1" \
  --api-key "{env:OPENROUTER_API_KEY}"

# Test provider
peekaboo config test-provider openrouter

# List models
peekaboo config models-provider openrouter

# List all providers
peekaboo config list-providers

# Remove provider
peekaboo config remove-provider openrouter
```

### Manage Credentials

```bash
# Add credential
peekaboo config set-credential OPENAI_API_KEY --value sk-...

# Validate credentials
peekaboo config validate-credential OPENAI_API_KEY

# Login (OAuth flow)
peekaboo config login openai
```

---

## Error Handling

### Configuration Validation

```swift
enum ConfigurationValidationError: LocalizedError {
    case invalidIdentifier(String)  // Empty provider ID
    case invalidName(String)        // Empty provider name
    case invalidURL(String)         // Malformed URL
    case invalidAPIKey(String)      // Empty API key
    case invalidHeaders(String)     // Malformed headers
    case invalidModels(String)      // Empty model names
}
```

### Decoding Errors

```swift
private func loadConfigurationFromPath(_ configPath: String) -> Configuration? {
    do {
        let data = try Data(contentsOf: URL(fileURLWithPath: configPath))
        let jsonString = String(data: data, encoding: .utf8) ?? ""
        let cleanedJSON = self.stripJSONComments(from: jsonString)
        let expandedJSON = self.expandEnvironmentVariables(in: cleanedJSON)
        
        let config = try JSONCoding.decoder.decode(Configuration.self, from: expandedData)
        return config
    } catch let error as DecodingError {
        switch error {
        case let .keyNotFound(key, context):
            let path = self.codingPathDescription(context)
            self.printWarning("JSON key not found '\(key.stringValue)' at path: \(path)")
        case let .typeMismatch(type, context):
            let path = self.codingPathDescription(context)
            self.printWarning("Type mismatch for type '\(type)' at path: \(path)")
        // ... more cases
        }
    } catch {
        self.printWarning("Failed to load configuration: \(error)")
    }
    return nil
}
```

---

## Debugging Configuration Issues

### Check File Permissions

```bash
# Should be 700 for directory
ls -ld ~/.peekaboo

# Should be 600 for credentials
ls -l ~/.peekaboo/credentials
```

### Validate JSONC Syntax

```bash
# Use jq to validate (after stripping comments)
peekaboo config show --json | jq .
```

### Trace Value Resolution

```bash
# Check which value is being used
peekaboo config show --effective

# Check environment variables
echo "PEEKABOO_AI_PROVIDERS=$PEEKABOO_AI_PROVIDERS"
echo "OPENAI_API_KEY=$OPENAI_API_KEY"
```

### Test Provider Connectivity

```bash
# Test custom provider
peekaboo config test-provider myprovider

# With verbose output
peekaboo config test-provider myprovider --verbose
```

---

## Integration Points

### CommandRuntime Integration

```swift
// Apps/CLI/Sources/PeekabooCLI/Commands/Base/CommandRuntime.swift
struct CommandRuntime {
    let configuration: Configuration
    let services: PeekabooServices
    
    init(
        configuration: Configuration,
        services: PeekabooServices)
    {
        TachikomaConfiguration.profileDirectoryName = ".peekaboo"
        // ... setup
    }
}

// How commands get configuration
struct SomeCommand: ParsableCommand {
    @RuntimeStorage var runtime: CommandRuntime?
    
    private var configuration: CommandRuntime.Configuration {
        self.runtimeOptions.makeConfiguration()
    }
}
```

### Tachikoma Integration

```swift
// AI provider setup in Tachikoma
TachikomaConfiguration.profileDirectoryName = ".peekaboo"

let provider = try AIConfiguration.fromEnvironment()
let model = try provider.getModel("gpt-5.1")
```

### Service Initialization

```swift
// PeekabooServices setup
let services = PeekabooServices()
services.installAgentRuntimeDefaults()

// Now MCP tools and ToolRegistry can resolve services
let automation = services.automation
let configManager = services.configManager
```

---

## Known Issues & Limitations

### Issue #70: Analyze Tool Ignores Config

**Problem:** The `analyze` MCP tool always uses `gpt-5.1`, ignoring configuration.

**Location:** Likely in `Core/PeekabooCore/Sources/PeekabooAgentRuntime/Tools/AnalyzeTool.swift`

**Fix needed:** Read from `ConfigurationManager` instead of hardcoding.

### macOS Tahoe Permissions

**Issue #51, #52:** CLI apps blocked from permissions on Tahoe.

**Workaround:** Use `.app` bundle instead of CLI.

**Root cause:** macOS Tahoe changes to permission system.

### Configuration Reload

**Limitation:** Changes to config.json don't reload during long-running processes.

**Impact:** Must restart daemon/MCP server after editing.

**Fix needed:** Add file watching and hot reload.

---

## Best Practices

### 1. Use Environment Variables for Secrets

```json
// ❌ Don't hardcode API keys in config.json
{
  "aiProviders": {
    "openaiApiKey": "sk-..."
  }
}

// ✅ Do use environment variable references
{
  "aiProviders": {
    "openaiApiKey": "${OPENAI_API_KEY}"
  }
}
```

### 2. Store Credentials Separately

```bash
# Use credentials file
peekaboo config set-credential OPENAI_API_KEY --value sk-...

# Or environment variables
export OPENAI_API_KEY=sk-...
```

### 3. Use Custom Providers for Flexibility

```bash
# Instead of hardcoding provider URLs
peekaboo config add-provider myprovider \
  --type openai \
  --base-url "https://api.mycompany.com/v1" \
  --api-key "{env:COMPANY_API_KEY}"
```

### 4. Test Before Saving

```bash
# Dry run to see changes
peekaboo config add-provider test --dry-run

# Test connectivity
peekaboo config test-provider myprovider

# Discover models
peekaboo config models-provider myprovider
```

### 5. Validate After Editing

```bash
# Always validate after manual edits
peekaboo config edit
# ... make changes ...
peekaboo config validate
```

---

## Future Enhancements

### 1. Configuration Hot Reload

```swift
// Add file watcher
class ConfigurationManager {
    private var configWatcher: DispatchSourceFileSystemObject?
    
    func startWatching() {
        // Monitor config.json for changes
        // Reload automatically
    }
}
```

### 2. Schema Validation

```swift
// Add JSON Schema validation
func validateConfiguration(_ config: Configuration) -> [ValidationError] {
    // Check required fields
    // Validate URL formats
    // Check provider IDs
    return errors
}
```

### 3. Configuration Inheritance

```json
{
  "extends": "/etc/peekaboo/base-config.json",
  "aiProviders": {
    // Override base config
  }
}
```

### 4. Provider Templates

```bash
# Pre-configured provider templates
peekaboo config add-provider --template openrouter
peekaboo config add-provider --template ollama
peekaboo config add-provider --template together
```

---

## Quick Reference

### File Locations
- Config: `~/.peekaboo/config.json`
- Credentials: `~/.peekaboo/credentials`
- Override dir: `PEEKABOO_CONFIG_DIR` env var

### Common Commands
```bash
peekaboo config init              # Create default config
peekaboo config show              # View config
peekaboo config show --effective  # View merged config
peekaboo config edit              # Edit in editor
peekaboo config validate          # Validate config
peekaboo config add-provider      # Add custom provider
peekaboo config test-provider     # Test connectivity
peekaboo config models-provider   # List available models
```

### Environment Variables
- `PEEKABOO_AI_PROVIDERS` - Provider list
- `OPENAI_API_KEY` - OpenAI key
- `ANTHROPIC_API_KEY` - Anthropic key
- `PEEKABOO_OLLAMA_BASE_URL` - Ollama endpoint

### Precedence Order
CLI args → Env vars → Credentials → Config file → Defaults

---

This deep dive provides complete coverage of Peekaboo's configuration management module. Use it as a reference for debugging, extending, or contributing to the configuration system.
