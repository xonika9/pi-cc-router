---
phase: 01-core-subprocess-bridge
plan: 03
subsystem: subprocess-bridge
tags:
  [
    typescript,
    cross-spawn,
    readline,
    subprocess,
    provider-registration,
    ndjson,
    vitest,
    tdd,
  ]

# Dependency graph
requires:
  - phase: 01-core-subprocess-bridge/01
    provides: "Wire protocol types (NdjsonMessage, ClaudeApiEvent) and prompt builder (buildPrompt, buildSystemPrompt)"
  - phase: 01-core-subprocess-bridge/02
    provides: "NDJSON stream parser (parseLine) and event bridge state machine (createEventBridge)"
provides:
  - "Process manager for subprocess spawn, stdin write, kill-after-result cleanup, stderr capture (src/process-manager.ts)"
  - "Provider orchestration function streamViaCli that ties all modules together (src/provider.ts)"
  - "Extension entry point with provider registration and startup validation (index.ts)"
  - "Complete working pi extension that registers as pi-claude-cli provider"
affects: [tool-handling, extended-thinking, mcp-proxy, platform-hardening]

# Tech tracking
tech-stack:
  added: []
  patterns:
    [
      cross-spawn-subprocess,
      kill-after-result,
      ndjson-stdin-write,
      readline-stdout-parse,
      abort-signal-handling,
      startup-validation,
    ]

key-files:
  created:
    - src/process-manager.ts
    - src/provider.ts
    - index.ts
    - tests/process-manager.test.ts
    - tests/provider.test.ts
  modified: []

key-decisions:
  - "Used cross-spawn for subprocess spawning (locked decision) for Windows .cmd shim compatibility"
  - "Kill-after-result with 2000ms grace period to handle CLI hang bug"
  - "Stdin stays open after writing user message (needed for control_response in Phase 2)"
  - "PassThrough streams in provider tests for readline.createInterface compatibility"

patterns-established:
  - "Subprocess lifecycle: spawnClaude -> writeUserMessage -> readline parse loop -> cleanupProcess"
  - "Startup validation: validateCliPresence (throws) + validateCliAuth (warns) at registration time"
  - "Provider registration: getModels('anthropic').map() for dynamic model catalog, streamSimple delegates to streamViaCli"
  - "Abort signal handling: addEventListener('abort') kills proc, removeEventListener in finally block"

requirements-completed: [PROV-01, PROV-03, PROC-01]

# Metrics
duration: 5min
completed: 2026-03-14
---

# Phase 1 Plan 03: Process Manager and Provider Wiring Summary

**Cross-platform subprocess manager with kill-after-result cleanup, provider registration via getModels('anthropic'), and streamViaCli orchestrator connecting prompt builder, NDJSON parser, and event bridge into a complete pi extension**

## Performance

- **Duration:** 5 min
- **Started:** 2026-03-14T03:46:51Z
- **Completed:** 2026-03-14T03:52:05Z
- **Tasks:** 3
- **Files modified:** 5

## Accomplishments

- Process manager implements all subprocess lifecycle functions: spawn with 12 CLI flags, NDJSON stdin write without closing, SIGKILL cleanup after 2000ms grace, stderr capture, CLI presence and auth validation
- streamViaCli orchestrates the full pipeline: buildPrompt -> buildSystemPrompt -> spawnClaude -> writeUserMessage -> readline NDJSON parse -> event bridge -> cleanupProcess, with abort signal and error handling
- Extension entry point registers pi-claude-cli provider with dynamically derived Anthropic models and startup validation
- 27 new tests (18 process-manager + 9 provider), bringing total to 84 tests across 5 test files, all passing with zero type errors

## Task Commits

Each task was committed atomically:

1. **Task 1 RED: Failing process manager tests** - `6a53f16` (test)
2. **Task 1 GREEN: Implement process manager** - `1a55854` (feat)
3. **Task 2 RED: Failing provider tests** - `88df8c1` (test)
4. **Task 2 GREEN: Implement provider and index** - `01da6ca` (feat)
5. **Task 3: Full test suite verification** - `d00bd72` (fix)

## Files Created/Modified

- `src/process-manager.ts` - Subprocess spawn, stdin write, kill-after-result cleanup, stderr capture, CLI validation
- `src/provider.ts` - streamViaCli orchestration connecting all modules with readline NDJSON parsing
- `index.ts` - Extension entry point with provider registration and startup validation
- `tests/process-manager.test.ts` - 18 unit tests for all process manager functions
- `tests/provider.test.ts` - 9 unit tests for provider registration and streamViaCli orchestration

## Decisions Made

- Used `cross-spawn` for subprocess spawning per locked decision, ensuring Windows `.cmd` shim compatibility
- Kill-after-result pattern with 2000ms grace period addresses known CLI hang bug (issues #25629, #21099)
- Stdin stays open after writing user message -- required for `control_response` messages in Phase 2 tool handling
- Used `PassThrough` streams in provider tests instead of plain `EventEmitter` for compatibility with `readline.createInterface`

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed TS2540 read-only property error in process-manager tests**

- **Found during:** Task 3 (full test suite verification)
- **Issue:** Mock ChildProcess object typed as `ChildProcess` made `killed` property read-only, causing `tsc --noEmit` to fail with TS2540
- **Fix:** Changed mock proc to use `any` type before casting, allowing the mock kill function to set `killed = true`
- **Files modified:** tests/process-manager.test.ts
- **Verification:** `tsc --noEmit` passes cleanly, all 84 tests pass
- **Committed in:** d00bd72 (Task 3 commit)

**2. [Rule 3 - Blocking] Provider test mock stdout incompatible with readline**

- **Found during:** Task 2 GREEN (provider tests failing)
- **Issue:** Mock stdout using plain `EventEmitter` was not compatible with `readline.createInterface` which requires a proper Readable stream
- **Fix:** Changed mock stdout from `EventEmitter` to `PassThrough` stream, and changed tests from `emit('data', ...)` to `write()`/`end()` pattern
- **Files modified:** tests/provider.test.ts
- **Verification:** All 9 provider tests pass
- **Committed in:** 01da6ca (Task 2 GREEN commit)

---

**Total deviations:** 2 auto-fixed (1 bug, 1 blocking)
**Impact on plan:** Both fixes necessary for test correctness. No scope creep.

## Issues Encountered

None beyond the type error and mock stream compatibility documented above.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- Phase 1 complete: all 6 source modules and 5 test files in place
- Extension is ready for end-to-end testing with pi (requires Claude CLI installed and authenticated)
- Foundation ready for Phase 2 (tool handling via control_request/control_response)
- Foundation ready for Phase 3 (extended thinking content blocks)
- Event bridge already logs warnings for tool_use and thinking blocks (Phase 2/3 extension points)

---

## Self-Check: PASSED

All 5 files verified present. All 5 commits verified in git log.

---

_Phase: 01-core-subprocess-bridge_
_Completed: 2026-03-14_
