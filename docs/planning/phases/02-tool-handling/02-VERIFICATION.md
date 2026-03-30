---
phase: 02-tool-handling
verified: 2026-03-14T11:35:00Z
status: passed
score: 8/8 must-haves verified
re_verification: false
---

# Phase 2: Tool Handling Verification Report

**Phase Goal:** Claude proposes tool calls that pi intercepts, translates, and executes natively -- the "Claude proposes, pi executes" loop works
**Verified:** 2026-03-14T11:35:00Z
**Status:** passed
**Re-verification:** No -- initial verification

## Goal Achievement

### Observable Truths (from ROADMAP.md Success Criteria)

| #   | Truth                                                                                                    | Status   | Evidence                                                                                                |
| --- | -------------------------------------------------------------------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------- |
| 1   | When Claude wants to read a file, pi receives a tool call with pi name `read` and translated arg `path`  | VERIFIED | `event-bridge.ts` maps via `mapClaudeToolNameToPi` + `translateClaudeArgsToPi` at block_stop            |
| 2   | Claude's built-in tool execution is denied via control protocol -- tools never execute inside subprocess | VERIFIED | `control-handler.ts` `handleControlRequest` always writes `behavior: "deny"` for non-MCP tools          |
| 3   | MCP-prefixed tool calls are allowed through the control protocol without denial                          | VERIFIED | `handleControlRequest` checks `tool_name.startsWith("mcp__")` and writes `behavior: "allow"`            |
| 4   | Pi receives `toolcall_start`, `toolcall_delta`, and `toolcall_end` events with correct names/args        | VERIFIED | `event-bridge.ts` emits all three events for `tool_use` content blocks; 9 event-bridge tests cover this |

**Score:** 4/4 ROADMAP success criteria verified

### Must-Have Truths (from PLAN frontmatter -- Plan 01)

| #   | Truth                                                                         | Status   | Evidence                                                                                                    |
| --- | ----------------------------------------------------------------------------- | -------- | ----------------------------------------------------------------------------------------------------------- |
| 1   | Claude tool name 'Read' maps to pi name 'read' and vice versa for all 6 tools | VERIFIED | `TOOL_MAPPINGS` array in `tool-mapping.ts` covers all 6; 8 name-mapping tests pass                          |
| 2   | Pi names 'find' AND 'glob' both map back to Claude's 'Glob'                   | VERIFIED | `PI_TO_CLAUDE_NAME["glob"] = "Glob"` added explicitly after loop; dedicated test passes                     |
| 3   | Claude arg 'file_path' maps to pi arg 'path' for Read/Write/Edit tools        | VERIFIED | `CLAUDE_TO_PI_ARGS` entries confirmed; 5 arg-translation tests pass                                         |
| 4   | Unknown tool names pass through unchanged in both directions                  | VERIFIED | `?? claudeName` / `?? piName` fallback in both mapping functions; passthrough tests pass                    |
| 5   | Unknown arguments pass through unchanged (not dropped)                        | VERIFIED | Translation iterates input keys and only renames known ones; "preserves unknown args" test passes           |
| 6   | Built-in tool control_requests are denied with correct message                | VERIFIED | Deny message constant `TOOL_EXECUTION_DENIED_MESSAGE` exported and included in response; 7 tests cover this |
| 7   | MCP-prefixed tool control_requests are allowed                                | VERIFIED | `mcp__` prefix check in `handleControlRequest`; 3 allow tests pass                                          |
| 8   | Control response includes matching request_id                                 | VERIFIED | `request_id: msg.request_id` in `ControlResponse` construction; dedicated test verifies this                |

**Score Plan 01:** 8/8 truths verified

### Must-Have Truths (from PLAN frontmatter -- Plan 02)

| #   | Truth                                                                                               | Status   | Evidence                                                                                                                                                       |
| --- | --------------------------------------------------------------------------------------------------- | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | When Claude streams tool_use block, pi receives toolcall_start, toolcall_delta, toolcall_end events | VERIFIED | Event bridge handles all three event types for `input_json_delta` deltas; 9 dedicated tests pass                                                               |
| 2   | toolcall_start fires immediately with mapped tool name on content_block_start                       | VERIFIED | `handleContentBlockStart` creates `TrackedToolBlock`, calls `mapClaudeToolNameToPi`, pushes event                                                              |
| 3   | toolcall_delta streams raw JSON argument fragments as they arrive                                   | VERIFIED | `handleContentBlockDelta` pushes `event.delta!.partial_json` unchanged as `delta` field                                                                        |
| 4   | toolcall_end contains fully parsed and argument-mapped ToolCall object                              | VERIFIED | `handleContentBlockStop` parses, calls `translateClaudeArgsToPi(block.claudeName, parsed)`; test confirms `{ path: "/foo.ts" }` for `{ file_path: "/foo.ts" }` |
| 5   | If JSON arg accumulation fails to parse, toolcall_end emits raw string instead of parsed object     | VERIFIED | `catch` branch sets `finalArgs = block.partialJson`; dedicated "raw string" test passes                                                                        |
| 6   | control_request messages from stdout are intercepted and responded to on stdin                      | VERIFIED | `provider.ts` line 121-123: `else if (msg.type === "control_request") { handleControlRequest(msg, proc!.stdin!); }`                                            |
| 7   | Built-in tools are denied, MCP tools are allowed                                                    | VERIFIED | Routing goes through `handleControlRequest` which implements deny/allow; 2 provider tests confirm                                                              |
| 8   | Prompt builder maps pi tool names and args back to Claude format for history replay                 | VERIFIED | `prompt-builder.ts` imports `mapPiToolNameToClaude` + `translatePiArgsToClaude`; 5 tests confirm                                                               |

**Score Plan 02:** 8/8 truths verified

---

### Required Artifacts

| Artifact                        | Expected                                                            | Status   | Details                                                                                                                                    |
| ------------------------------- | ------------------------------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `src/tool-mapping.ts`           | Single-source mapping table, 5 exports                              | VERIFIED | 112 lines, exports `TOOL_MAPPINGS`, `mapClaudeToolNameToPi`, `mapPiToolNameToClaude`, `translateClaudeArgsToPi`, `translatePiArgsToClaude` |
| `src/control-handler.ts`        | Control protocol handler, exports `handleControlRequest` + constant | VERIFIED | 74 lines, exports both `handleControlRequest` and `TOOL_EXECUTION_DENIED_MESSAGE`; also exports `MCP_PREFIX`                               |
| `src/types.ts`                  | `ClaudeControlRequest` with `request_id` and nested `request`       | VERIFIED | Lines 25-33 confirm wire protocol format with `request_id` and `request.{ subtype, tool_name, input }`                                     |
| `src/event-bridge.ts`           | Tool_use content block handling with toolcall events                | VERIFIED | `TrackedToolBlock` interface, `tool_use` branch in all three content block handlers                                                        |
| `src/provider.ts`               | control_request routing to `handleControlRequest`                   | VERIFIED | Line 121-123: `else if (msg.type === "control_request")` routes to handler                                                                 |
| `src/prompt-builder.ts`         | Pi-to-Claude reverse mapping via `mapPiToolNameToClaude`            | VERIFIED | Line 13 imports both functions; `contentToText` and `buildPrompt` use them                                                                 |
| `tests/tool-mapping.test.ts`    | Unit tests for name mapping and arg translation                     | VERIFIED | 30 tests covering all 6 tools, asymmetry, passthrough, unknown args                                                                        |
| `tests/control-handler.test.ts` | Unit tests for control protocol deny/allow logic                    | VERIFIED | 17 tests covering deny, allow, response format, malformed input                                                                            |
| `tests/event-bridge.test.ts`    | Tests for tool_use streaming (Phase 2 additions)                    | VERIFIED | 9 new tests in `tool_use content block streaming` describe block                                                                           |
| `tests/provider.test.ts`        | Tests for control_request routing                                   | VERIFIED | 2 new tests: routing test + stream-continues-after-control test                                                                            |
| `tests/prompt-builder.test.ts`  | Tests for reverse name/arg mapping in history replay                | VERIFIED | 5 tests in `tool name and argument reverse mapping` describe block                                                                         |

---

### Key Link Verification

**Plan 01 key links:**

| From                     | To                    | Via                                               | Status     | Details                                                                                  |
| ------------------------ | --------------------- | ------------------------------------------------- | ---------- | ---------------------------------------------------------------------------------------- |
| `src/control-handler.ts` | `src/tool-mapping.ts` | imports `mapClaudeToolNameToPi` for MCP detection | NOT LINKED | Control handler uses `startsWith("mcp__")` directly -- does NOT import from tool-mapping |
| `src/control-handler.ts` | `src/types.ts`        | imports `ClaudeControlRequest` type               | VERIFIED   | Line 9: `import type { ClaudeControlRequest } from "./types";`                           |

**Note on first key link:** The PLAN specified that `control-handler.ts` would import `mapClaudeToolNameToPi` "for MCP prefix detection." In practice, the implementation uses `tool_name.startsWith(MCP_PREFIX)` directly where `MCP_PREFIX = "mcp__"` is a constant defined in `control-handler.ts` itself. This is a correct and intentional deviation -- the MCP prefix check does not need the tool-mapping module. The behavior specified in the truth ("MCP-prefixed tool control_requests are allowed") is fully implemented and tested. This is a documentation artifact in the PLAN, not a code defect.

**Plan 02 key links:**

| From                    | To                       | Via                                                             | Status   | Details                                                                                        |
| ----------------------- | ------------------------ | --------------------------------------------------------------- | -------- | ---------------------------------------------------------------------------------------------- |
| `src/event-bridge.ts`   | `src/tool-mapping.ts`    | imports `mapClaudeToolNameToPi` and `translateClaudeArgsToPi`   | VERIFIED | Line 11: `import { mapClaudeToolNameToPi, translateClaudeArgsToPi } from "./tool-mapping.js";` |
| `src/provider.ts`       | `src/control-handler.ts` | imports `handleControlRequest`, routes control_request messages | VERIFIED | Line 20: import; line 121-123: routing in `rl.on("line")` callback                             |
| `src/prompt-builder.ts` | `src/tool-mapping.ts`    | imports `mapPiToolNameToClaude` and `translatePiArgsToClaude`   | VERIFIED | Line 13: `import { mapPiToolNameToClaude, translatePiArgsToClaude } from "./tool-mapping.js";` |

---

### Requirements Coverage

| Requirement | Source Plan  | Description                                                                       | Status    | Evidence                                                                                         |
| ----------- | ------------ | --------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------ |
| STRM-03     | 02-01, 02-02 | Extension sends `control_response` messages to deny or allow tool execution       | SATISFIED | `handleControlRequest` writes JSON control_response to stdin; 17 tests confirm                   |
| TOOL-01     | 02-01, 02-02 | Extension denies all built-in tool execution requests via control protocol        | SATISFIED | Non-`mcp__` tools always get `behavior: "deny"`; wired in provider.ts                            |
| TOOL-02     | 02-01, 02-02 | Extension allows `mcp__` prefixed tool execution via control protocol             | SATISFIED | `startsWith("mcp__")` check returns `behavior: "allow"`; wired in provider.ts                    |
| TOOL-03     | 02-01, 02-02 | Extension maps tool names bidirectionally between Claude names and pi equivalents | SATISFIED | `TOOL_MAPPINGS` array + 4 functions cover all 6 tools bidirectionally                            |
| TOOL-04     | 02-01, 02-02 | Extension translates tool arguments between Claude and pi formats                 | SATISFIED | `translateClaudeArgsToPi` / `translatePiArgsToClaude` applied in event-bridge and prompt-builder |

**All 5 phase requirements satisfied. No orphaned requirements found.**

Traceability note: REQUIREMENTS.md lists STRM-03 and TOOL-01 through TOOL-04 as mapping to Phase 2 with status "Complete" -- consistent with this verification.

---

### Anti-Patterns Found

| File                    | Line | Pattern                         | Severity | Impact                                                                                                                                                |
| ----------------------- | ---- | ------------------------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/prompt-builder.ts` | 118  | Comment uses word "placeholder" | Info     | No impact -- comment describes the runtime fallback string `[${block.type}]` for unknown block types. The implementation is complete and intentional. |

No stub implementations, empty handlers, or TODO markers found in Phase 2 source files.

---

### Human Verification Required

None. All Phase 2 behaviors are verifiable programmatically:

- Tool name and argument translation is pure data transformation (fully covered by unit tests)
- Control protocol request/response is wire-format JSON (fully covered by unit tests)
- Event streaming is synchronous mock-based (fully covered by unit tests)
- TypeScript compilation is clean (tsc --noEmit produces no output)

---

### Commit Verification

All 8 task commits from both SUMMARYs verified in git history:

| Commit    | Description                                                                            |
| --------- | -------------------------------------------------------------------------------------- |
| `ad652a6` | test(02-01): add failing tests for tool name/arg mapping                               |
| `d324b03` | feat(02-01): implement tool mapping module                                             |
| `fce2ac4` | test(02-01): add failing tests for control handler + type update                       |
| `c11ff96` | feat(02-01): implement control protocol handler                                        |
| `66fe3c5` | test(02-02): add failing tests for tool_use content block streaming                    |
| `5edc4ee` | feat(02-02): add tool_use content block handling to event bridge                       |
| `8365e1e` | test(02-02): add failing tests for control_request routing and tool name mapping       |
| `80beb6a` | feat(02-02): wire control handler into provider, add reverse mapping to prompt builder |

---

### Test Suite Results

- **Total tests:** 150 (all pass)
- **Phase 2 additions:** 47 tests (Plan 01) + 16 tests (Plan 02) = 63 new tests
- **Phase 1 regression:** All 87 Phase 1 tests continue to pass
- **TypeScript:** `tsc --noEmit` exits clean (zero errors)

---

### Gaps Summary

No gaps. All must-haves verified. The single key-link discrepancy (control-handler.ts not importing from tool-mapping.ts for MCP detection) is a PLAN documentation artifact -- the behavior it was meant to ensure ("MCP tools allowed") is implemented correctly via an equivalent direct prefix check. Goal achievement is not affected.

---

_Verified: 2026-03-14T11:35:00Z_
_Verifier: Claude (gsd-verifier)_
