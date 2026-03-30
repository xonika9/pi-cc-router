---
status: resolved
trigger: "Custom tool MCP proxy architecture is fundamentally flawed — MCP tools bypass control protocol"
created: 2026-03-14
updated: 2026-03-14
resolution: "Phase 4 implementation reverted. Architectural redesign needed."
---

## Root Cause

MCP tools bypass the Claude CLI control protocol entirely. `--allowedTools` pre-approves them at the CLI permission layer, and they never generate `control_request` messages. This means our "Claude proposes, pi executes" pattern cannot work for MCP tools — the control handler never gets a chance to deny them for pi to execute.

Without MCP, Claude has no tool schemas for custom tools and can't propose calls (ToolSearch loop). With MCP, Claude calls tools but results go to the MCP server, not pi. Either way, pi can't execute custom tools.

## The Architectural Problem

1. Claude Code only knows how to call **built-in tools** (Read/Write/Edit/Bash/Grep/Glob — schemas built in) and **MCP tools** (schemas from MCP server)
2. Built-in tools go through the control protocol → we deny → pi executes. **This works.**
3. MCP tools bypass the control protocol → Claude executes via MCP → pi never gets involved. **This doesn't work.**
4. Custom tools mentioned in system prompt text are NOT callable — Claude has no schema for them

## What Was Tried (All Failed)

| Attempt                             | Why It Failed                                                                                     |
| ----------------------------------- | ------------------------------------------------------------------------------------------------- |
| MCP proxy with denial stubs         | MCP server returns stub, pi never executes. Claude gets useless "handled by host" response        |
| Allow ToolSearch in control handler | ToolSearch resolves deferred tools, not MCP tools. Custom tools aren't deferred                   |
| Suppress ToolSearch events from pi  | Prevented pi error but Claude still couldn't call custom tools                                    |
| --allowedTools for MCP tools        | Permission works, but bypasses control protocol — pi can't intercept                              |
| System prompt instructions          | Claude acknowledges MCP tool names but still won't call them (or calls via MCP, gets denial stub) |
| Remove MCP entirely                 | Without schemas, Claude can't propose tool calls. ToolSearch loop                                 |

## Correct Architecture (for future phase)

The MCP server needs **IPC back to the extension** to get real tool results from pi:

1. MCP server receives tool call from Claude
2. MCP server sends tool name + args to extension via IPC (temp file, named pipe, or socket)
3. Extension executes tool via pi's runtime
4. Extension sends result back to MCP server via IPC
5. MCP server returns real result to Claude

This is significantly more complex than the denial-stub approach but is the only way to bridge MCP execution to pi's tool runtime.

## Resolution

- Phase 4 implementation reverted to pre-Phase-4 state (187 tests, 8 files)
- All MCP proxy files removed (mcp-server.ts, mcp-server.js, mcp-config.ts)
- @modelcontextprotocol/sdk dependency removed
- Phase 4 needs re-planning with IPC-based MCP architecture
- test-tool-extension.ts kept for future testing
