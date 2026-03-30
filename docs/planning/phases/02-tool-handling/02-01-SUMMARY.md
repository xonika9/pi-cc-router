---
phase: 02-tool-handling
plan: 01
subsystem: tool-handling
tags: [tool-mapping, control-protocol, ndjson, stream-json, mcp]

# Dependency graph
requires:
  - phase: 01-core-subprocess-bridge
    provides: types.ts with NdjsonMessage union, stream-parser for NDJSON parsing
provides:
  - Single-source tool mapping table (TOOL_MAPPINGS) with bidirectional name and argument translation
  - Control protocol handler (handleControlRequest) with deny/allow logic for control_request messages
  - Updated ClaudeControlRequest type with request_id and nested request structure
affects:
  [
    02-tool-handling plan 02,
    prompt-builder history replay,
    provider stream processing,
  ]

# Tech tracking
tech-stack:
  added: []
  patterns:
    [
      single-source mapping table with derived lookups,
      data-driven argument renaming with passthrough,
    ]

key-files:
  created:
    - src/tool-mapping.ts
    - src/control-handler.ts
    - tests/tool-mapping.test.ts
    - tests/control-handler.test.ts
  modified:
    - src/types.ts
    - tests/stream-parser.test.ts

key-decisions:
  - "Single-source TOOL_MAPPINGS array with derived lookup maps (no separate per-direction tables)"
  - "Argument translation starts from input object and only renames specific keys (never drops unknown args)"
  - "handleControlRequest accepts NodeJS.WritableStream (not ChildProcess) for testability and decoupling"
  - "Malformed control_request messages log to console.error and return false (graceful degradation)"

patterns-established:
  - "Data-driven mapping: define mappings as data, derive lookups programmatically"
  - "PassThrough streams in tests for writable stream mocking"

requirements-completed: [TOOL-03, TOOL-04, STRM-03, TOOL-01, TOOL-02]

# Metrics
duration: 4min
completed: 2026-03-14
---

# Phase 2 Plan 1: Tool Mapping and Control Handler Summary

**Single-source bidirectional tool mapping table and control protocol handler with deny/allow logic for 6 built-in tools and MCP-prefixed tools**

## Performance

- **Duration:** 4 min
- **Started:** 2026-03-14T16:14:50Z
- **Completed:** 2026-03-14T16:18:39Z
- **Tasks:** 2
- **Files modified:** 6

## Accomplishments

- Created tool-mapping.ts with TOOL_MAPPINGS array and 4 exported translation functions (mapClaudeToolNameToPi, mapPiToolNameToClaude, translateClaudeArgsToPi, translatePiArgsToClaude)
- Created control-handler.ts that denies built-in tools and allows mcp\_\_ prefixed tools via wire protocol control_response messages
- Updated ClaudeControlRequest type from flat format to verified wire protocol format (request_id + nested request object)
- 47 new tests (30 tool-mapping + 17 control-handler), full suite 134 tests green, tsc clean

## Task Commits

Each task was committed atomically:

1. **Task 1: Create tool mapping module** - `ad652a6` (test: RED) + `d324b03` (feat: GREEN)
2. **Task 2: Update types and create control handler** - `fce2ac4` (test: RED) + `c11ff96` (feat: GREEN)

_Note: TDD tasks have separate test and implementation commits_

## Files Created/Modified

- `src/tool-mapping.ts` - Single-source mapping table with bidirectional name and argument translation
- `src/control-handler.ts` - Control protocol handler for deny/allow logic
- `src/types.ts` - Updated ClaudeControlRequest to wire protocol format
- `tests/tool-mapping.test.ts` - 30 tests covering all mappings, asymmetry, and passthrough
- `tests/control-handler.test.ts` - 17 tests covering deny/allow, response format, malformed input
- `tests/stream-parser.test.ts` - Updated control_request test to use new wire format

## Decisions Made

- Used single-source TOOL_MAPPINGS array instead of separate per-direction lookup tables (improves on reference project pattern)
- Argument translation iterates input keys and renames only known ones, preserving all unknown args (avoids Pitfall 5 from research)
- handleControlRequest takes NodeJS.WritableStream instead of ChildProcess for testability -- caller passes proc.stdin!
- Malformed control_request messages (missing request_id or request) are handled gracefully with console.error and return false

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Updated stream-parser test to match new ClaudeControlRequest wire format**

- **Found during:** Task 2 (type update)
- **Issue:** Existing stream-parser.test.ts used the old flat control_request format which no longer matches the updated type definition
- **Fix:** Updated test to use the verified wire protocol format with request_id and nested request object
- **Files modified:** tests/stream-parser.test.ts
- **Verification:** Full test suite passes (134 tests)
- **Committed in:** fce2ac4 (Task 2 test commit)

---

**Total deviations:** 1 auto-fixed (1 bug fix)
**Impact on plan:** Necessary correction to keep existing tests consistent with type changes. No scope creep.

## Issues Encountered

None

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- tool-mapping.ts is ready for import by event-bridge.ts (Plan 02) for tool_use content block handling
- control-handler.ts is ready for import by provider.ts (Plan 02) for control_request stream processing
- All 134 existing tests remain green with the updated ClaudeControlRequest type

## Self-Check: PASSED

All 6 files verified present. All 4 task commits verified in git log.

---

_Phase: 02-tool-handling_
_Completed: 2026-03-14_
