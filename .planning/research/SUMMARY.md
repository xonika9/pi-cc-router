# Research Summary: pi-claude-cli

**Domain:** CLI subprocess LLM provider extension
**Researched:** 2026-03-13
**Overall confidence:** MEDIUM-HIGH

## Executive Summary

This extension bridges pi's custom provider interface (`streamSimple`) with the Claude Code CLI subprocess, enabling pi users to leverage a Claude Pro/Max subscription instead of an API key. The architecture is well-defined thanks to a proven reference implementation (`claude-agent-sdk-pi`) that validates the overall pattern, though this project replaces the Claude Agent SDK transport with a direct subprocess using the stream-json wire protocol.

The core challenge is translating between two event systems: Claude's raw API streaming events (`content_block_start/delta/stop`, `message_start/delta/stop`) and pi's `AssistantMessageEventStream` events (`text_start/delta/end`, `toolcall_start/delta/end`, `thinking_start/delta/end`). The bidirectional control protocol (`control_request`/`control_response`) handles tool approval/denial -- denying built-in tools so pi executes them, while allowing MCP-prefixed tools so Claude executes custom tool calls.

The main architectural unknown is the custom tool MCP proxy. The reference project uses the SDK's `createSdkMcpServer()`, which transparently handles tool exposure. Without the SDK, we need to implement an equivalent stdio MCP server. Two approaches exist (schema-only vs delegating proxy), and the right choice depends on how Claude behaves when MCP tool calls are denied -- this needs experimental validation in Phase 4.

The stateless subprocess model (fresh `claude -p` per request, full history replayed) is the only proven pattern. While it incurs token overhead from replaying conversation history, this is acceptable because the subscription model has no per-token cost. Persistent sessions would be more efficient but are unproven with this protocol and add significant complexity.

## Key Findings

**Stack:** TypeScript extension for pi, using Node.js `child_process.spawn()`, readline for NDJSON parsing, and `@modelcontextprotocol/sdk` for the MCP proxy server.

**Architecture:** Six major components -- ExtensionEntry, PromptBuilder, ProcessManager, StreamParser/EventBridge (stream translation), ToolRouter (control protocol), ToolMapper (name/arg translation), and McpToolProxy (custom tool exposure). Build order follows dependency chain: subprocess communication first, then tool handling, then custom tool proxy.

**Critical pitfall:** Double tool execution -- if both the MCP proxy allows a tool call AND pi executes it separately, the tool runs twice with potential side effects. The control protocol must be the single decision point.

## Implications for Roadmap

Based on research, suggested phase structure:

1. **Core Subprocess Bridge** - Get text streaming working end-to-end
   - Addresses: Provider registration, subprocess lifecycle, basic streaming
   - Avoids: Tool complexity before proving the subprocess protocol works

2. **Built-in Tool Handling** - Add tool denial + mapping
   - Addresses: Tool call interception, name/arg translation, pi tool execution
   - Avoids: Custom tool complexity (MCP proxy) before built-in tools work

3. **Extended Thinking + Usage** - Complete event coverage
   - Addresses: Thinking tokens, usage metrics, cost tracking
   - Avoids: Shipping incomplete event support

4. **Custom Tool MCP Proxy** - Expose pi custom tools to Claude
   - Addresses: MCP server implementation, tool schema exposure
   - Avoids: Attempting this before the control protocol is proven (Phase 2)

5. **Platform Hardening** - Cross-platform, error recovery, edge cases
   - Addresses: Windows compatibility, process cleanup, crash recovery
   - Avoids: Premature optimization before functional correctness

**Phase ordering rationale:**
- Phase 1 proves the fundamental subprocess communication works
- Phase 2 depends on Phase 1 (needs subprocess to send control_requests)
- Phase 3 is independent of Phase 2 but shares infrastructure
- Phase 4 depends on Phase 2 (needs control protocol for MCP tool decisions)
- Phase 5 is a polish pass after all features work

**Research flags for phases:**
- Phase 4: Likely needs deeper research (MCP proxy approach decision, Claude behavior with denied MCP tools)
- Phase 1: Standard patterns, unlikely to need research (spawn, readline, NDJSON)
- Phase 2: May need research (exact argument mapping schemas for current pi/Claude versions)

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Node.js child_process, readline, MCP SDK are well-documented standard tools |
| Features | HIGH | Reference project validates the feature set; pi provider API is well-documented |
| Architecture | MEDIUM-HIGH | Component boundaries clear; MCP proxy approach needs experimental validation |
| Pitfalls | MEDIUM | Control protocol pitfalls documented; platform-specific issues need testing |

## Gaps to Address

- Exact argument mapping schemas need verification against current pi and Claude CLI versions
- Image content block handling in stream-json input is undocumented
- MCP proxy approach (schema-only vs delegating) needs experimental validation
- Windows subprocess behavior (`shell: true`, `.cmd` extension) needs platform testing
- Prompt length limits and conversation history truncation strategy unclear
