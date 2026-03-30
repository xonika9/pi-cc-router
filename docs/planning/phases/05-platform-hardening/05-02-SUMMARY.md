---
phase: 05-platform-hardening
plan: 02
subsystem: infra
tags: [images, base64, content-blocks, anthropic-api, history-replay, HIST-02]

# Dependency graph
requires:
  - phase: 01-core-subprocess-bridge
    provides: prompt-builder.ts (buildPrompt, userContentToText), process-manager.ts (writeUserMessage)
  - phase: 05-platform-hardening
    plan: 01
    provides: process-manager.ts (forceKillProcess, registerProcess), provider.ts streamEnded guard pattern
provides:
  - buildPrompt returns string | AnthropicContentBlock[] (image passthrough for final user message)
  - translateImageBlock (pi-ai to Anthropic API format conversion)
  - buildFinalUserContent (structured content blocks for final user message)
  - userContentToText inserts placeholder text for images instead of skipping
  - writeUserMessage accepts string | any[] content
  - Console.warn once per request for placeholder image count
affects: [provider.ts consumers, future image handling improvements]

# Tech tracking
tech-stack:
  added: []
  patterns:
    [
      AnthropicContentBlock type alias,
      pi-to-Anthropic image translation,
      placeholder image count with module-level counter,
    ]

key-files:
  created: []
  modified:
    [
      src/prompt-builder.ts,
      src/process-manager.ts,
      tests/prompt-builder.test.ts,
      tests/process-manager.test.ts,
    ]

key-decisions:
  - "buildPrompt return type is string | AnthropicContentBlock[] -- only returns array when final user message has valid images"
  - "Invalid image blocks (missing data/mimeType) fall back to placeholder text, not error"
  - "Module-level placeholderImageCount counter reset per buildPrompt call for simple tracking"
  - "buildCustomToolResultPrompt path always returns string (no image passthrough in custom tool result flow)"
  - "writeUserMessage type broadened to string | any[] -- JSON.stringify handles both natively"

patterns-established:
  - "Image translation pattern: pi { type: image, data, mimeType } -> Anthropic { type: image, source: { type: base64, media_type, data } }"
  - "Dual-return buildPrompt: string for text-only, ContentBlock[] for image passthrough"
  - "Placeholder image pattern: insert readable text + console.warn with count"

requirements-completed: [HIST-02]

# Metrics
duration: 4min
completed: 2026-03-15
---

# Phase 5 Plan 2: Image History Replay Summary

**Image passthrough translating pi-ai image blocks to Anthropic API format in final user messages, with placeholder text and console.warn for non-final historical images**

## Performance

- **Duration:** 4 min
- **Started:** 2026-03-15T05:55:43Z
- **Completed:** 2026-03-15T05:59:25Z
- **Tasks:** 1
- **Files modified:** 4

## Accomplishments

- Final user message images translated from pi-ai format to Anthropic API format (base64 content blocks)
- Non-final user message images replaced with readable placeholder text
- Console.warn emitted once per request with placeholder image count (warn-don't-block pattern)
- Backward compatible: buildPrompt returns string when no images, writeUserMessage handles both types
- All 248 tests pass with zero regressions

## Task Commits

Each task was committed atomically (TDD: RED then GREEN):

1. **Task 1: Update buildPrompt for image passthrough and writeUserMessage for array content** - RED `69215ad` / GREEN `1f1e3b0`

## Files Created/Modified

- `src/prompt-builder.ts` - Added AnthropicContentBlock type, translateImageBlock, buildFinalUserContent, contentHasImages helpers; updated userContentToText for placeholders; buildPrompt returns string | ContentBlock[]
- `src/process-manager.ts` - Updated writeUserMessage signature to accept string | any[]
- `tests/prompt-builder.test.ts` - Updated image skip test to expect placeholder; added 7 new image passthrough tests
- `tests/process-manager.test.ts` - Added 2 new writeUserMessage tests for string and array content

## Decisions Made

- buildPrompt returns `string | AnthropicContentBlock[]` -- only returns array when final user message has valid images with both `data` and `mimeType` fields
- Invalid image blocks (missing data/mimeType) gracefully fall back to placeholder text, never error
- Module-level `placeholderImageCount` counter reset per buildPrompt call -- simple approach avoids changing userContentToText return type
- `buildCustomToolResultPrompt` path always returns string (no image passthrough needed for custom tool result turns)
- `writeUserMessage` type broadened to `string | any[]` -- JSON.stringify handles both natively, no additional logic needed

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None -- plan executed smoothly with no issues.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- HIST-02 (image history replay) is complete
- All Phase 5 requirements satisfied (PROC-02, PROC-03, PROC-04 from Plan 01; HIST-02 from Plan 02)
- Phase 5 is fully complete, ready for Phase 6 or UAT

## Self-Check: PASSED

All 5 files verified present. All 2 commit hashes verified in git log.

---

_Phase: 05-platform-hardening_
_Completed: 2026-03-15_
