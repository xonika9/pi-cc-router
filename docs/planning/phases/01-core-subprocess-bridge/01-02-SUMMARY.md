---
phase: 01-core-subprocess-bridge
plan: 02
subsystem: streaming
tags: [typescript, ndjson, event-bridge, state-machine, vitest, tdd]

# Dependency graph
requires:
  - phase: 01-core-subprocess-bridge/01
    provides: "Wire protocol types (NdjsonMessage, ClaudeApiEvent, TrackedContentBlock)"
provides:
  - "NDJSON stream parser with resilient error handling (src/stream-parser.ts)"
  - "Event bridge state machine bridging Claude API events to pi AssistantMessageEventStream (src/event-bridge.ts)"
affects: [01-03, tool-handling, extended-thinking]

# Tech tracking
tech-stack:
  added: []
  patterns:
    [
      resilient-ndjson-parsing,
      event-bridge-state-machine,
      stop-reason-mapping,
      tdd-red-green,
    ]

key-files:
  created:
    - src/stream-parser.ts
    - src/event-bridge.ts
    - tests/stream-parser.test.ts
    - tests/event-bridge.test.ts
  modified: []

key-decisions:
  - "Used pi-ai's AssistantMessage type directly instead of local redefinition for type safety"
  - "Event bridge uses array index mapping between TrackedContentBlock and output.content for multi-block support"
  - "parseLine logs malformed JSON to console.error for debugging but never throws"

patterns-established:
  - "Resilient parsing: trim, check prefix, try/catch JSON.parse, validate object shape"
  - "Event bridge factory pattern: createEventBridge returns handleEvent/getOutput interface"
  - "Stop reason mapping: end_turn->stop, max_tokens->length, tool_use->toolUse, unknown->stop"
  - "Content block tracking: TrackedContentBlock[] parallel to output.content[] for index correlation"

requirements-completed: [STRM-01, STRM-02]

# Metrics
duration: 6min
completed: 2026-03-14
---

# Phase 1 Plan 02: Stream Parser and Event Bridge Summary

**NDJSON stream parser that never throws plus event bridge state machine translating Claude API content_block/message events to pi text_start/delta/end/done events with usage tracking**

## Performance

- **Duration:** 6 min
- **Started:** 2026-03-14T03:38:18Z
- **Completed:** 2026-03-14T03:44:05Z
- **Tasks:** 2
- **Files modified:** 4

## Accomplishments

- Resilient NDJSON parser that handles valid JSON, debug noise, malformed lines, empty lines, and non-object JSON without throwing
- Event bridge state machine that correctly produces text_start, text_delta, text_end, and done pi events from Claude API streaming events
- Usage data captured from message_start and message_delta events with calculateCost integration
- Stop reason mapping (end_turn->stop, max_tokens->length, tool_use->toolUse) and done event reason alignment
- Tool_use and thinking content blocks handled with warnings (deferred to Phase 2/3)
- 43 total tests across both modules (24 stream-parser, 19 event-bridge), all passing with tsc --noEmit clean

## Task Commits

Each task was committed atomically:

1. **Task 1 RED: Failing stream parser tests** - `a7b0d80` (test)
2. **Task 1 GREEN: Implement NDJSON stream parser** - `5807611` (feat)
3. **Task 2 RED: Failing event bridge tests** - `1574f2c` (test)
4. **Task 2 GREEN: Implement event bridge state machine** - `ca7b8b2` (feat)

## Files Created/Modified

- `src/stream-parser.ts` - Resilient NDJSON line parser (parseLine function)
- `src/event-bridge.ts` - Event bridge state machine (createEventBridge factory)
- `tests/stream-parser.test.ts` - 24 unit tests for stream parser
- `tests/event-bridge.test.ts` - 19 unit tests for event bridge

## Decisions Made

- Used pi-ai's `AssistantMessage` and `TextContent` types directly rather than redefining locally, ensuring type compatibility with stream push events
- Event bridge uses parallel array tracking: `TrackedContentBlock[]` for internal state (with Claude's `index`) alongside `output.content[]` for the final message, correlated by array position
- parseLine logs parse failures to `console.error` rather than silently swallowing -- aids debugging without crashing the pipeline

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed content type mismatch with pi-ai's AssistantMessage**

- **Found during:** Task 2 GREEN (event bridge implementation)
- **Issue:** Initial implementation defined a local `AssistantMessage` interface with `content: Array<{ type: string; text: string }>` which was incompatible with pi-ai's `AssistantMessage` requiring `content: (TextContent | ThinkingContent | ToolCall)[]` -- the `type: string` vs `type: "text"` literal caused tsc errors
- **Fix:** Imported `AssistantMessage` and `TextContent` directly from `@mariozechner/pi-ai` and used `as const` assertions for literal types
- **Files modified:** src/event-bridge.ts
- **Verification:** `npx tsc --noEmit` passes cleanly, all 43 tests still pass
- **Committed in:** ca7b8b2 (Task 2 GREEN commit)

---

**Total deviations:** 1 auto-fixed (1 bug)
**Impact on plan:** Essential for type safety with pi-ai's type system. No scope creep.

## Issues Encountered

None beyond the type mismatch documented above.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- Stream parser and event bridge ready for integration in provider handler (Plan 01-03)
- Both modules independently testable with comprehensive test coverage
- Event bridge prepared for Phase 2/3 extension (tool_use/thinking blocks logged as warnings)

---

## Self-Check: PASSED

All 4 files verified present. All 4 commits verified in git log.

---

_Phase: 01-core-subprocess-bridge_
_Completed: 2026-03-14_
