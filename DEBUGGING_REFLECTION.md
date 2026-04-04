# Debugging Reflection: Custom Provider Integration

## Chain of Thoughts

The debugging followed a **systematic layer-by-layer approach**: start from configuration loading, trace through provider registration, follow the request pipeline, and finally examine response parsing. Each fix revealed the next bug hidden deeper in the stack.

## Key Design Points

**1. Lazy Loading Pitfall**: Multiple modules assumed credentials/registries were pre-loaded. The fix: explicit loading at service initialization boundaries, not at access points.

**2. Protocol Flexibility**: SSE spec allows `data:` or `data: ` (space optional). Bailian's omission broke our strict prefix check. Robust parsers handle both.

**3. Enum Extensibility**: Anthropic-compatible providers may extend the protocol (thinking blocks). Enums should gracefully ignore or pass-through unknown content types rather than failing silently.

**4. Provider Abstraction**: Custom providers need first-class treatment throughout the stack—not as an afterthought. The provider parsing logic must check custom registries before falling back to built-ins.

## Success Points

**Incremental Verification**: After each fix, immediate testing isolated failures. Testing with both `curl` (bypassing Swift) and the actual agent command distinguished networking bugs from parsing bugs.

**Cross-Command Coverage**: Agent worked but `see --analyze` didn't—different service classes. Fixed once, applied everywhere.

**Environment Awareness**: Xcode Swift vs. swift.org Swift have different API availability. System proxy blocked Swift URLSession but not curl. Environment matters as much as code.

**The Breakthrough**: Thinking content + SSE prefix were the final blockers. Once both fixed, the custom provider worked perfectly across all commands.

**Total**: 6 bugs fixed, 9 commits, ~200 lines changed. The key was refusing to accept "it works with env vars" as sufficient—config file support required following every code path end-to-end.

---

*Generated from custom provider integration bug fix session - April 2026*
