---
phase: 02-tool-handling
plan: 02
subsystem: tool-handling
tags: [tool-use, content-block, streaming, control-protocol, event-bridge, prompt-builder]

# Dependency graph
requires:
  - phase: 01-core-subprocess-bridge
    provides: event-bridge.ts, provider.ts, prompt-builder.ts base modules
  - phase: 02-tool-handling plan 01
    provides: tool-mapping.ts (TOOL_MAPPINGS, mapClaudeToolNameToPi, translateClaudeArgsToPi), control-handler.ts (handleControlRequest), updated ClaudeControlRequest type
provides:
  - Event bridge tool_use content block handling with toolcall_start/delta/end events
  - Provider control_request routing to handleControlRequest via proc.stdin
  - Prompt builder pi-to-Claude reverse tool name and argument mapping for history replay
affects: [phase-03 extended-thinking, phase-04 mcp-proxy, phase-05 history]

# Tech tracking
tech-stack:
  added: []
  patterns: [TrackedToolBlock discriminated union for tool_use block state, partial JSON accumulation with incremental parse attempts, raw string fallback for unparseable tool arguments]

key-files:
  created: []
  modified:
    - src/event-bridge.ts
    - src/provider.ts
    - src/prompt-builder.ts
    - src/types.ts
    - tests/event-bridge.test.ts
    - tests/provider.test.ts
    - tests/prompt-builder.test.ts

key-decisions:
  - "TrackedToolBlock with claudeName field for arg translation at block_stop (pi name stored separately)"
  - "ToolCall.arguments typed as Record but cast via `as ToolCall` to support raw string fallback"
  - "TrackedContentBlock.type narrowed to 'text' | 'thinking' (tool_use now uses TrackedToolBlock)"
  - "Partial JSON parse attempts during delta (update args on success, keep previous on failure)"

patterns-established:
  - "Discriminated union narrowing: TrackedBlock = TrackedContentBlock | TrackedToolBlock for type-safe content block handling"
  - "Incremental JSON parsing: accumulate partial_json, try parse after each delta, update state only on success"

requirements-completed: [STRM-03, TOOL-01, TOOL-02, TOOL-03, TOOL-04]

# Metrics
duration: 7min
completed: 2026-03-14
---

# Phase 2 Plan 2: Event Bridge, Provider, and Prompt Builder Tool Wiring Summary

**Tool_use content block streaming with toolcall events, control_request routing via provider, and bidirectional tool name/arg mapping in prompt builder history replay**

## Performance

- **Duration:** 7 min
- **Started:** 2026-03-14T16:21:30Z
- **Completed:** 2026-03-14T16:28:35Z
- **Tasks:** 2
- **Files modified:** 7

## Accomplishments
- Event bridge handles tool_use content blocks with toolcall_start/delta/end streaming events, mapping Claude tool names to pi names and translating arguments at block_stop
- Provider routes control_request messages to handleControlRequest with proc.stdin, enabling deny/allow responses in the subprocess control protocol
- Prompt builder applies reverse mapping (pi-to-Claude) for tool names and arguments when serializing tool call history, ensuring Claude receives its own naming conventions in history replay
- 16 new tests (9 event-bridge, 2 provider, 5 prompt-builder), full suite 150 tests green, tsc clean

## Task Commits

Each task was committed atomically:

1. **Task 1: Add tool_use content block handling to event bridge** - `66fe3c5` (test: RED) + `5edc4ee` (feat: GREEN)
2. **Task 2: Wire control handler into provider and add reverse mapping to prompt builder** - `8365e1e` (test: RED) + `80beb6a` (feat: GREEN)

_Note: TDD tasks have separate test and implementation commits_

## Files Created/Modified
- `src/event-bridge.ts` - Added TrackedToolBlock type, tool_use content block handling with toolcall_start/delta/end events, partial JSON accumulation, argument mapping
- `src/types.ts` - Narrowed TrackedContentBlock.type to "text" | "thinking" (tool_use blocks use TrackedToolBlock)
- `src/provider.ts` - Added control_request routing to handleControlRequest with proc.stdin
- `src/prompt-builder.ts` - Added mapPiToolNameToClaude and translatePiArgsToClaude for history replay
- `tests/event-bridge.test.ts` - 9 new tests for tool_use streaming (replaced old "logs warning" test)
- `tests/provider.test.ts` - 2 new tests for control_request handling
- `tests/prompt-builder.test.ts` - 5 new tests for reverse name/arg mapping, updated existing tests for Claude naming

## Decisions Made
- Stored original Claude name as `claudeName` on TrackedToolBlock for argument translation at block_stop (since block.name is already the pi name)
- Used `as ToolCall` cast for toolcall_end construction because pi-ai's ToolCall.arguments is typed as `Record<string, any>` but we intentionally emit raw strings on parse failure
- Narrowed TrackedContentBlock.type from `"text" | "tool_use" | "thinking"` to `"text" | "thinking"` since tool_use blocks now use the dedicated TrackedToolBlock type -- fixes TypeScript discriminated union narrowing
- Partial JSON is parsed after each delta for incremental updates; on failure, previous parsed args are kept (graceful degradation)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed TrackedContentBlock type to enable discriminated union narrowing**
- **Found during:** Task 1 (GREEN phase, tsc check)
- **Issue:** TrackedContentBlock had `type: "text" | "tool_use" | "thinking"` which overlaps with TrackedToolBlock's `type: "tool_use"`, preventing TypeScript from narrowing the union in `if (block.type === "tool_use")` checks
- **Fix:** Changed TrackedContentBlock.type to `"text" | "thinking"` since tool_use blocks now use the dedicated TrackedToolBlock type
- **Files modified:** src/types.ts
- **Verification:** tsc --noEmit passes with zero errors
- **Committed in:** 5edc4ee (Task 1 implementation commit)

**2. [Rule 1 - Bug] Fixed syntax error in provider.ts control_request routing**
- **Found during:** Task 2 (GREEN phase, test run)
- **Issue:** Initial edit created a mismatched brace structure by replacing a comment with an `else if` clause that had an extra closing brace
- **Fix:** Corrected the brace structure to properly chain the `else if` for control_request handling
- **Files modified:** src/provider.ts
- **Verification:** All provider tests pass, tsc clean
- **Committed in:** 80beb6a (Task 2 implementation commit)

---

**Total deviations:** 2 auto-fixed (2 bug fixes)
**Impact on plan:** Both auto-fixes necessary for TypeScript correctness and syntactic validity. No scope creep.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Phase 2 is now complete: full "Claude proposes, pi executes" tool loop works end-to-end
- Event bridge emits toolcall events with pi tool names, provider handles control protocol, prompt builder replays history with Claude naming
- Ready for Phase 3 (Extended Thinking + Usage Metrics) and Phase 4 (MCP Proxy)

## Self-Check: PASSED

All 7 files verified present. All 4 task commits verified in git log.

---
*Phase: 02-tool-handling*
*Completed: 2026-03-14*
