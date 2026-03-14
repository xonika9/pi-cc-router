# Phase 2: Tool Handling - Context

**Gathered:** 2026-03-13
**Status:** Ready for planning

<domain>
## Phase Boundary

Intercept Claude's tool proposals via the stream-json control protocol, deny built-in tool execution (pi executes natively), allow `mcp__` prefixed tools (Claude executes its own MCP server tools), map tool names and arguments bidirectionally between Claude and pi, and emit toolcall streaming events. Custom tool MCP proxy (exposing pi tools to Claude) is Phase 4. Extended thinking and usage metrics are Phase 3.

</domain>

<decisions>
## Implementation Decisions

### Control Protocol (Deny/Allow)
- Deny ALL non-MCP tools via `control_response` with `behavior: "deny"`
- Allow `mcp__` prefixed tools via `control_response` with `behavior: "allow"` — these are Claude's own MCP server tools that pi can't execute
- Deny message: `"Tool execution is unavailable in this environment."` (matches reference project)
- Deny each `control_request` individually as it arrives — no batching, immediate response per request
- Allowed MCP tool calls still emit toolcall events to pi (pi can display them in the UI even though it didn't execute them)

### Unknown/Unmapped Tool Handling
- Pass through unchanged — if Claude proposes a tool not in the 6-tool mapping table, emit toolcall events with the original Claude tool name and args unmodified
- Matches reference project behavior: no warning, no drop, no error for unknown tools
- Pi decides what to do with unknown tool names on its side

### Tool Name Mapping
- Bidirectional mapping for 6 known tools: Read↔read, Write↔write, Edit↔edit, Bash↔bash, Grep↔grep, Glob↔find
- Single source of truth mapping table — derive both directions (Claude→pi and pi→Claude) from one definition
- Handle the glob/find asymmetry: Claude's Glob maps to pi's `find`; pi's `find` AND `glob` both map back to Claude's `Glob` (matches reference project)
- Centralized mapping table as a data structure (not inline per-tool switch/case) — easy to update when either system changes

### Argument Translation
- Only translate known renamed arguments + pass everything else through unchanged (matches reference project)
- Known renames: `file_path`↔`path` (Read/Write/Edit), `old_string`↔`oldText` and `new_string`↔`newText` (Edit), `head_limit`↔`limit` (Grep)
- Arguments with the same name in both systems pass through untouched
- Centralized mapping table — arg renames defined alongside tool name mappings in the same data structure

### Tool Call Streaming Events
- `toolcall_start` fires immediately on `content_block_start` (as soon as tool name is known, before args arrive)
- `toolcall_delta` streams raw JSON argument fragments as they arrive from `content_block_delta` events
- `toolcall_end` emits after `content_block_stop` with fully accumulated and parsed args object (with name/arg mapping applied)
- If accumulated JSON args fail to parse: emit `toolcall_end` with the raw unparsed string instead of a parsed object — don't silently drop the event

### Claude's Discretion
- Internal buffering strategy for accumulating JSON argument fragments
- Exact control_response JSON structure (beyond the decided behavior/message fields)
- Error handling for malformed control_request messages
- Timing/ordering of toolcall events relative to text events in multi-content-block responses

</decisions>

<specifics>
## Specific Ideas

- Reference project (`claude-agent-sdk-pi`) is the primary pattern reference — follow its mapping tables, deny-all approach, and passthrough behavior for unknowns
- Reference project's deny message constant: `TOOL_EXECUTION_DENIED_MESSAGE = "Tool execution is unavailable in this environment."`
- Reference project denies ALL tools (including MCP) since it uses the Agent SDK — we diverge by allowing `mcp__` tools since we use the CLI control protocol directly
- Reference project uses two separate lookup tables (`SDK_TO_PI_TOOL_NAME` / `PI_TO_SDK_TOOL_NAME`) — we improve on this with a single-source approach

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- None yet — Phase 1 not executed. Phase 1 plans define the NDJSON parser and event bridge that this phase extends

### Established Patterns
- Phase 1 plans establish: readline-based NDJSON parsing, `AssistantMessageEventStream` event emission, subprocess stdin/stdout communication
- Control protocol handler will extend the existing stream parser from Phase 1

### Integration Points
- Stream parser (Phase 1) — add control_request detection and control_response writing
- Event bridge (Phase 1) — add toolcall_start/delta/end event emission alongside existing text events
- Prompt builder (Phase 1) — add pi→Claude tool name/arg mapping for history replay with prior tool results

</code_context>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 02-tool-handling*
*Context gathered: 2026-03-13*
