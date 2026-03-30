# Phase 4: Custom Tool MCP Proxy - Context

**Gathered:** 2026-03-14
**Updated:** 2026-03-14 (third revision — break-early architecture)
**Status:** Ready for planning

<domain>
## Phase Boundary

Expose pi's custom tools (non-built-in) to Claude via a minimal stdio MCP server so Claude can propose calling them. Prevent Claude CLI from executing tools directly — pi handles all tool execution. Ensure existing user-side MCP configurations (`.mcp.json`, `~/.claude.json`) continue working.

</domain>

<architectural_findings>

## Empirical Findings (2026-03-14 testing session)

### Discovery 1: control_request was never firing

The `--permission-mode dontAsk` flag causes the CLI to auto-approve built-in tools and auto-deny MCP tools WITHOUT sending `control_request` messages. Our control handler (`src/control-handler.ts`) was dead code — never called. Built-in tools were being auto-executed by Claude CLI directly.

### Discovery 2: --permission-prompt-tool stdio

The Agent SDK uses `--permission-prompt-tool stdio` to enable `control_request`/`control_response` exchange. Without this flag, the CLI never sends `control_request`. With it (and default permission mode, NOT dontAsk), MCP tools DO get `control_request`. Built-in tools still auto-approve.

### Discovery 3: Break-early pattern

The reference project (`claude-agent-sdk-pi`) handles tool execution by breaking out of the stream loop at `message_stop` when `sawToolCall` is true, then killing the subprocess. This prevents the CLI from auto-executing tools (execution happens AFTER `message_stop`). Pi then executes the tools itself.

### Discovery 4: MCP server only needs tools/list

In dontAsk mode, MCP tools are auto-denied — the CLI never calls `tools/call`. In default mode with break-early, the subprocess is killed before `tools/call`. Either way, the MCP server only needs to implement `tools/list` for schema discovery.

### Discovery 5: User-side MCPs work through control_request

With `--permission-prompt-tool stdio` (default mode), user-side MCP tools get `control_request`. We allow them → Claude executes them via their MCP servers. This only applies in the "don't break" path.

</architectural_findings>

<decisions>
## Implementation Decisions

### Architecture: Break-Early + Schema-Only MCP

**Core principle:** Pi controls all tool execution. Claude proposes tools but never executes them (except user-side MCPs).

**CLI flag changes:**

- Remove `--permission-mode dontAsk`
- Add `--permission-prompt-tool stdio` (enables control_request for MCP tools)

**Break-early logic (in provider.ts):**
At `message_stop`, check tool_use blocks seen in the current turn:

- Any built-in tool OR `mcp__custom-tools__*` → break early, kill subprocess, pi executes all
- Only user MCPs (`mcp__*` but NOT `mcp__custom-tools__*`) → don't break, allow via control_request, Claude executes them

**Control handler changes:**

- Only reached in the "don't break" path (all tools are user MCPs)
- Allow all `control_request` in this path (we already decided not to break)

**MCP server:**

- Minimal stdio JSON-RPC script (~30 lines, no SDK dependency)
- Only implements `initialize` and `tools/list` (reads schemas from temp JSON file)
- `tools/call` never reached (break-early kills subprocess first)
- Plain `.cjs` file (Claude CLI spawns with `node`)

**Schema delivery:**

- Extension calls `pi.getAllTools()` lazily on first request
- Filters out 6 built-in tools (`read`, `write`, `edit`, `bash`, `grep`, `find`)
- Writes custom tool schemas to temp JSON file
- MCP config points to schema server with temp file path as arg

### Confirmed (carried forward)

- `pi.getAllTools()` for custom tool discovery — lazy, not at load time
- Filter out 6 built-in tools
- TypeBox schemas pass through as JSON Schema — no conversion needed
- `--mcp-config` flag to register MCP server with subprocess
- Don't pass `--strict-mcp-config` (CONF-01) — user MCP configs load
- Warn-don't-block if MCP setup fails
- Strip `mcp__custom-tools__` prefix from tool call names (MCP-02)

### Invalidated (from previous attempts)

- ~~`--permission-mode dontAsk`~~ → Use default mode (no permission-mode flag)
- ~~control_request deny pattern~~ → Break-early is the deny mechanism
- ~~MCP server needs to execute tools~~ → MCP server is schema-only
- ~~`@modelcontextprotocol/sdk` dependency~~ → Raw JSON-RPC, no SDK needed
- ~~HTTP/in-process MCP transport~~ → Simple stdio, separate process
- ~~IPC for tool results~~ → Not needed, MCP never executes tools

</decisions>

<specifics>
## Specific Ideas

- MCP server `.cjs` file reads tool schemas from a temp JSON file path passed as CLI arg
- Break-early uses the same `shouldStopEarly` pattern as the reference project
- The event bridge already processes tool_use content blocks — just needs MCP prefix stripping
- Tool mapping already handles unknown tools (pass-through) — extend for MCP prefix

</specifics>

<code_context>

## Existing Code Changes Needed

### process-manager.ts

- Remove `--permission-mode dontAsk` from spawnClaude args
- Add `--permission-prompt-tool stdio`
- Add `--mcp-config` option for custom tools MCP server

### provider.ts

- Add break-early logic: track tool_use blocks, at message_stop check if should break
- Kill subprocess on break-early
- Handle control_request for user MCPs (allow path)

### control-handler.ts

- Simplify: only reached for user MCPs in "don't break" path
- Allow everything (we already decided not to break)

### tool-mapping.ts

- Add `mcp__custom-tools__` prefix stripping
- Map stripped names back to pi tool names

### event-bridge.ts

- Handle MCP-prefixed tool_use blocks (strip prefix in toolcall events)

### New: mcp-schema-server.cjs

- Minimal stdio JSON-RPC server
- Reads tool schemas from temp file
- Implements initialize + tools/list only

### New: mcp-config.ts

- Write custom tool schemas to temp JSON file
- Generate MCP config JSON pointing to schema server
- Cleanup temp files

</code_context>

<deferred>
## Deferred Ideas

- Built-in tool interception via SDK hook protocol (if break-early proves insufficient)
- Preventing built-in auto-execution in mixed built-in + user MCP turns (edge case)

</deferred>

---

_Phase: 04-custom-tool-mcp-proxy_
_Context gathered: 2026-03-14_
_Updated: 2026-03-14 — third revision after empirical testing of CLI permission behavior_
