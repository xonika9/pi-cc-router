# Phase 5: Platform Hardening - Context

**Gathered:** 2026-03-14
**Updated:** 2026-03-15 (Phase 4 architectural changes documented)
**Status:** Ready for re-planning (Phase 4 changed provider.ts significantly)

<domain>
## Phase Boundary

Make the extension work reliably across Windows, macOS, and Linux with proper error handling, subprocess lifecycle management, and image history support. Covers requirements PROC-02 (already satisfied), PROC-03, PROC-04, and HIST-02.

</domain>

<decisions>
## Implementation Decisions

### Error surfacing
- Include stderr output and exit code in error events — developers can debug, non-technical users can report
- Buffer stderr, only surface it when the process exits abnormally (not real-time to console)
- If partial text was already streamed before a crash, keep the partial response and append an error event at the end — don't discard useful output
- If images must fall back to placeholder: "[An image was shared here but could not be included]"
- Console.warn once per request when images fall back to placeholder (not per image) — follows existing warn-don't-block pattern

### Timeout policy
- Use an inactivity timeout (no stdout output for N seconds), not a total request timeout — long responses are fine as long as tokens keep flowing
- Hardcoded default duration, not configurable — keep it simple, can add configurability later if users ask
- When timeout fires: kill subprocess and emit error event with clear message
- **Research note for timeout duration:** In vanilla Claude Code, subagents can run 5-20+ minutes with zero output on the main process stdout. In pi-claude-cli's stateless architecture, built-in tools are denied so Claude can't spawn subagents — each request is one prompt→one response. However, researcher should validate whether extended thinking phases produce continuous thinking_delta events or have silent gaps that could false-trigger an inactivity timeout.

### Image history replay
- Attempt actual image passthrough: translate pi-ai image blocks (`{ type: "image", data, mimeType }`) to Anthropic API format (`{ type: "image", source: { type: "base64", media_type, data } }`) and send as array content blocks instead of flattened text
- This requires changing `buildPrompt()` to return structured content (array of content blocks) instead of a plain string for user messages that contain images, and updating `writeUserMessage()` to send array content in the NDJSON message
- Fallback to placeholder text "[An image was shared here but could not be included]" only if stream-json rejects array content blocks
- Console.warn once per request when N images fall back to placeholder — follows warn-don't-block pattern

### Process lifecycle
- Platform-aware kill strategy: taskkill /F on Windows, SIGKILL on Unix — ensures no orphans on any platform
- On abort signal (user cancels): immediate force-kill, no grace period — user wants it stopped now
- Global process registry: keep a Set of active child processes, kill all on extension deactivate or process exit — prevents orphaned claude processes
- Grace period after result event: reduce from 2000ms to 500ms — we have everything we need from the result, brief buffer for edge-case stdout flushing

### Claude's Discretion
- Exact wording of error messages for unknown failures (no stderr, non-zero exit code)
- Exact inactivity timeout duration (recommended: 120-300 seconds based on analysis)
- Implementation details of the process registry (Set vs WeakRef, cleanup timing)
- Platform detection approach for kill strategy

</decisions>

<specifics>
## Specific Ideas

- User observed that in Claude Code, subagent operations can take 5-20+ minutes with zero visible output in the main terminal — only the subagent's expanded view shows activity. In pi-claude-cli's architecture this shouldn't apply (built-in tools denied, no subagent spawning), but it informed the decision to use inactivity timeout rather than total request timeout, and to have the researcher validate silent gap behavior during extended thinking.
- PROC-02 (cross-spawn for Windows .cmd shim resolution) is already satisfied — `process-manager.ts` already imports and uses `cross-spawn`.
- The current abort handler uses SIGTERM (`provider.ts:71`) which doesn't work reliably on Windows — needs to be updated to use the platform-aware kill.

</specifics>

<phase4_changes>
## Critical: Phase 4 Changed provider.ts Architecture

Phase 4 (Custom Tool MCP Proxy) significantly changed `provider.ts` and related files. Plans written before Phase 4 reference outdated code patterns. Key changes:

### 1. stream.end() moved from event bridge to provider
`handleMessageStop` in `event-bridge.ts` is now a **no-op** — it does NOT push the `done` event or call `stream.end()`. The provider pushes `done` and calls `stream.end()` **after the readline close promise resolves** (async). This was required because pushing `done` synchronously in `handleMessageStop` prevented pi from executing tools.

**Implication for Phase 5:** Any `streamEnded` guard or error path refactoring must account for the fact that `stream.end()` happens after `await rl.on("close")`, not inside the event bridge. Error paths that push errors + call `stream.end()` must coordinate with the post-readline done/end logic.

### 2. Break-early logic in provider.ts
Provider tracks `sawBuiltInOrCustomTool` (via `isPiKnownClaudeTool()`) during stream processing. At `message_stop`, if true: sets `broken = true`, calls `proc.kill("SIGKILL")`, calls `rl.close()`, returns early.

**Implication for Phase 5:** The `broken` flag must be checked in ALL error/cleanup paths. When `broken` is true, process errors from SIGKILL should be ignored (already guarded: `if (broken) return` in the error handler). Any new error handlers or close handlers must also check `broken`.

### 3. proc.on("error") guard for intentional SIGKILL
The process error handler has `if (broken) return` to ignore errors from intentional SIGKILL (break-early). Without this, SIGKILL triggers an error event that would push an error to pi's stream after the done event.

**Implication for Phase 5:** The `endStreamWithError` helper (if introduced) must check `broken` before pushing errors.

### 4. CLI flags changed
- `--permission-mode dontAsk` was **removed**
- `--permission-prompt-tool stdio` was **added** (enables control_request for MCP tools)
- `--mcp-config` is conditionally passed (suppressed when custom tool results are pending)

**Implication for Phase 5:** Any references to `--permission-mode dontAsk` in plans are outdated. The CLI args are now: `-p --input-format stream-json --output-format stream-json --verbose --include-partial-messages --no-session-persistence --model {id} --permission-prompt-tool stdio` plus optional `--mcp-config`, `--append-system-prompt`, `--effort`.

### 5. MCP config suppression in provider
`hasPendingCustomToolResult(messages)` checks if the last messages are toolResult for custom tools. If true, `--mcp-config` is not passed to prevent Claude from re-calling tools via MCP instead of using the result.

### 6. New files from Phase 4
- `src/mcp-schema-server.cjs` — minimal stdio JSON-RPC MCP server
- `src/mcp-config.ts` — custom tool discovery and MCP config generation
- `src/tool-mapping.ts` — added `isPiKnownClaudeTool()`, `isCustomToolName()`, `CUSTOM_TOOLS_MCP_PREFIX`
- `src/control-handler.ts` — denies `mcp__custom-tools__*`, allows everything else

### 7. Known race condition (display issue)
Pushing `done` asynchronously (after readline closes) causes pi to render custom tool calls as text instead of blue/green boxes. Pushing synchronously breaks tool execution. This is a known tradeoff — functionality over display. Phase 5 should NOT attempt to "fix" this by moving `done` back to `handleMessageStop`.

</phase4_changes>

<code_context>
## Existing Code Insights

### Reusable Assets
- `process-manager.ts`: `cleanupProcess()` — needs grace period reduction (2000ms → 500ms) and platform-aware kill
- `process-manager.ts`: `captureStderr()` — already buffers stderr, returns getter function
- `provider.ts`: abort signal handler — needs update from SIGTERM to immediate force-kill
- `provider.ts`: process `error` event handler — has `if (broken) return` guard for break-early SIGKILL

### Established Patterns
- Warn-don't-block: `validateCliAuth()` logs warning and continues — image drop warning follows same pattern
- Error events: `provider.ts` pushes `{ type: "error", reason: "error", error: message }` — all new error paths should follow this shape
- Subprocess lifecycle managed in `provider.ts` `streamViaCli()` async IIFE
- Break-early pattern: track tool_use blocks, kill subprocess at message_stop, push done after readline closes
- `broken` flag: set before `rl.close()`, checked in error handlers to ignore intentional SIGKILL errors

### Integration Points
- `process-manager.ts`: Add platform-aware kill function, update `cleanupProcess()` grace period
- `provider.ts`: Add inactivity timeout, subprocess exit/close handlers, process registry integration — must coordinate with break-early logic and async done/end pattern
- `prompt-builder.ts`: `userContentToText()` — image blocks currently skipped, needs placeholder logic. Note: `buildCustomToolResultPrompt()` also exists for custom tool result turns
- `index.ts`: Process registry cleanup on extension deactivate (new integration point). Note: `ensureMcpConfig` pattern already exists here for MCP config lifecycle

</code_context>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 05-platform-hardening*
*Context gathered: 2026-03-14*
