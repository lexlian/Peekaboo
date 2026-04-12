# GitHub Issues Tracker - Known Issues

*Last updated: 2026-03-31*  
*Source: https://github.com/steipete/Peekaboo/issues*

## Summary

- **Open Issues/PRs:** 20
- **Critical Blocks:** Permissions (Tahoe), Bridge crashes, Element coordinate mapping
- **Version Affected:** Primarily 3.0.0-beta3 and beta4

---

## Critical Issues (High Priority)

### #76: Peekaboo.app crashes on start
- **URL:** https://github.com/steipete/Peekaboo/issues/76
- **Status:** Open
- **Description:** App crashes intermittently on launch, requires multiple attempts
- **Error Log:** Contains bundle info for v3.0.0-beta3 (1)
- **Impact:** Makes macOS app unusable for some users
- **Workaround:** Retry launching multiple times

### #52: Swift continuation leak causes see/image/permissions commands to hang on macOS Tahoe
- **URL:** https://github.com/steipete/Peekaboo/issues/52
- **Status:** Open
- **Environment:** macOS 26.2 (Tahoe beta)
- **Description:** Commands hang indefinitely due to Swift continuation leak
- **Impact:** Core functionality broken on Tahoe
- **Related:** #51 (Tahoe permissions)

### #51: macOS Tahoe permissions issue
- **URL:** https://github.com/steipete/Peekaboo/issues/51
- **Status:** Open
- **Description:** macOS Tahoe bug prevents CLI apps from getting permissions, only full macOS apps work
- **Impact:** SSH/service usage blocked
- **Workaround:** None currently (reported as macOS bug)

### #63: `peekaboo click --on elem_XX` coordinate mapping issue
- **URL:** https://github.com/steipete/Peekaboo/issues/63
- **Status:** Open
- **Description:** Click lands at incorrect position vs annotated screenshot element labels
- **Impact:** Element-based clicking unreliable
- **Workaround:** Use coordinate-based clicks as fallback

---

## Permission & Automation Issues

### #77: peekaboo no permission in openclaw
- **URL:** https://github.com/steipete/Peekaboo/issues/77
- **Status:** Open
- **Description:** Permission errors when running within OpenClaw environment
- **Evidence:** Screenshots show permission denial errors

### #73: AppleEvents Automation prompt never appears (kTCCServiceAppleEvents denied)
- **URL:** https://github.com/steipete/Peekaboo/issues/73
- **Status:** Open
- **Environment:** macOS 26.2, Peekaboo 3.0.0-beta3
- **Description:** Automation permission prompt never shows, access denied silently
- **Bridge Status:** Using local (in-process) bridge
- **Impact:** Cannot automate apps requiring AppleEvents

### #79: Oneliner wrong on Homepage
- **URL:** https://github.com/steipete/Peekaboo/issues/79
- **Status:** Open
- **Description:** Documented example `peekaboo "Open Safari..."` fails with "Unknown command" error
- **Impact:** Documentation misleading for new users
- **Fix Needed:** Update homepage example or fix agent command parsing

---

## Screen Capture Issues

### #67: image --app "Google Chrome" captures blank helper window
- **URL:** https://github.com/steipete/Peekaboo/issues/67
- **Status:** Open
- **Description:** Captures blank helper window instead of main Chrome browser window
- **Impact:** Cannot capture Chrome windows reliably
- **Workaround:** Use window-specific capture or screen capture mode

### #66: MCP or CLI `image` tool captures at 1x resolution instead of retina (2x)
- **URL:** https://github.com/steepete/Peekaboo/issues/66
- **Status:** Open
- **Description:** Output is 1x resolution despite Retina display
- **Expected:** Should capture at 2x when `--retina` flag used or by default on Retina displays
- **Impact:** Low-quality screenshots for AI analysis

---

## AI/Provider Issues

### #70: analyze tool ignores configured model, hardcodes gpt-5.1
- **URL:** https://github.com/steipete/Peekaboo/issues/70
- **Status:** Open
- **Description:** `analyze` MCP tool always uses `gpt-5.1`, ignoring `aiProviders` config and `PEEKABOO_AI_PROVIDERS` env
- **Impact:** Cannot use alternative models for analysis
- **Fix:** Respect configuration in Responses API calls

### #60: Support GLM-4V model normalized coordinates (0-1000 range)
- **URL:** https://github.com/steipete/Peekaboo/issues/60
- **Status:** Open
- **Description:** GLM-4V models return bounding boxes in 0-1000 normalized range instead of pixels
- **Impact:** Coordinate conversion fails for GLM models
- **Fix Needed:** Add normalized coordinate detection and conversion

### #58: Adding support for LLM proxies or LiteLLM
- **URL:** https://github.com/steipete/Peekaboo/issues/58
- **Status:** Open
- **Feature Request:** Support custom LLM proxies with custom headers/base URLs
- **Use Case:** LiteLLM, local proxies, enterprise gateways
- **Example:** `https://localhost:3333/1/openai/responses?api-version=2025...`

---

## Build & Dependency Issues

### #56: Package resolution failure - ElevenLabsKit revision mismatch
- **URL:** https://github.com/steipete/Peekaboo/issues/56
- **Status:** Open
- **Description:** Swift PM fails with ElevenLabsKit revision mismatch
- **Error:** Dependency resolution conflict
- **Impact:** Cannot build projects depending on Peekaboo

### #59: macOS 14 (Sonoma) support regression in v3.0.0-beta2+
- **URL:** https://github.com/steipete/Peekaboo/issues/59
- **Status:** Open
- **Description:** v3.0.0-beta2+ requires macOS 15+, breaking Sonoma support
- **Impact:** macOS 14 users cannot upgrade
- **Fix:** Restore macOS 14 compatibility or document minimum version clearly

---

## Release & Distribution Issues

### #57: Release v3.0.0-beta4 to GitHub (and Homebrew tap)
- **URL:** https://github.com/steipete/Peekaboo/issues/57
- **Status:** Open
- **Description:** Version bumped to beta4 in code but no GitHub release or Homebrew update
- **Impact:** Users cannot access beta4 via package managers
- **Action Needed:** Create GitHub release, update Homebrew tap

---

## Documentation & Enhancement Issues

### #100: 🤝 Lina Assistant - Collaboration Inquiry
- **URL:** https://github.com/steipete/Peekaboo/issues/100
- **Status:** Open
- **Type:** Spam/Collaboration request
- **Description:** AI assistant promotion from OpenClaw

### #99: support custom ai providers in image analyze (PR)
- **URL:** https://github.com/steipete/Peekaboo/pull/99
- **Status:** Open PR
- **Related:** #70 (analyze tool model config)

### #98: docs: add agent skill for Peekaboo CLI (PR)
- **URL:** https://github.com/steipete/Peekaboo/pull/98
- **Status:** Open PR
- **Type:** Documentation

### #97: docs: Add subprocess integration guide with permission workarounds (PR)
- **URL:** https://github.com/steipete/Peekaboo/pull/97
- **Status:** Open PR
- **Type:** Documentation

### #96: feat(cli): add completions (PR)
- **URL:** https://github.com/steipete/Peekaboo/pull/96
- **Status:** Open PR
- **Type:** Feature - shell completions

---

## Cross-Reference with CHANGELOG

### Recently Fixed (from CHANGELOG.md)
- ✅ Test runs hermeticity after MCP Swift SDK 0.11 updates
- ✅ macOS settings Google/Gemini and Grok provider surfacing
- ✅ MCP `list` / `see` text output improvements
- ✅ Agent tool response handling for MCP resource content
- ✅ CLI credential writes to config/profile directory
- ✅ Remote `peekaboo see` element detection timeout handling
- ✅ Screen recording permission check reliability
- ✅ Coordinate click fail-fast when target app not frontmost

### Known Issues in v3.0.0-beta4 (from CHANGELOG)
Check changelog for beta4 known issues section (mentioned in README line 15).

---

## Debugging Priority Order

When debugging issues, prioritize in this order:

1. **Crashers** - #76 (app crashes)
2. **Platform blocks** - #52, #51 (Tahoe compatibility)
3. **Core automation failures** - #73, #63 (permissions, coordinate mapping)
4. **Capture quality** - #67, #66 (wrong window, resolution)
5. **Configuration ignored** - #70 (model config)
6. **Build blockers** - #56, #59 (dependencies, OS support)

---

## Issue Templates for New Bugs

When creating new issues, include:

```markdown
## Environment
- **Peekaboo version:** `peekaboo --version`
- **macOS version:** `sw_vers`
- **Install method:** Homebrew / Source / Release binary
- **Bridge status:** `peekaboo bridge status`

## Description
[Clear description of the issue]

## Steps to Reproduce
1. `peekaboo <command> <args>`
2. [Second step]
3. [Expected vs actual behavior]

## Error Output
```
[Full error output with --verbose flag]
```

## Permissions
```bash
peekaboo permissions status
```

## What I've Tried
- [ ] Granted Screen Recording permission
- [ ] Granted Accessibility permission
- [ ] Restarted affected apps
- [ ] Tried with local build
- [ ] Checked bridge status
```

---

## Monitoring Script

Check for new issues:

```bash
#!/bin/bash
# Check open issues count
curl -s "https://api.github.com/repos/steipete/Peekaboo/issues?state=open" | \
  python3 -c "import sys,json; issues=json.load(sys.stdin); print(f'Open: {len(issues)}')"
```

---

## Related Documentation

- `DEBUGGING_ASSISTANT.md` - Quick debugging reference
- `CODEBASE_NAVIGATION.md` - Code structure guide
- `docs/permissions.md` - Permission troubleshooting
- `CHANGELOG.md` - Recent fixes and known issues
