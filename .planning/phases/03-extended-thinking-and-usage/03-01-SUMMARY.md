---
phase: 03-extended-thinking-and-usage
plan: 01
subsystem: config
tags: [thinking-effort, cli-flags, opus-model-detection, effort-mapping]

# Dependency graph
requires:
  - phase: 01-core-subprocess-bridge
    provides: spawnClaude subprocess spawning, streamViaCli provider orchestration
provides:
  - thinking-config module with effort mapping tables and Opus detection
  - effort flag wiring from pi SimpleStreamOptions through provider to subprocess
  - CliEffortLevel type for CLI --effort flag values
affects: [03-02 verification, future model additions]

# Tech tracking
tech-stack:
  added: []
  patterns: [effort-mapping-tables, opus-model-detection-via-includes]

key-files:
  created:
    - src/thinking-config.ts
    - tests/thinking-config.test.ts
  modified:
    - src/process-manager.ts
    - src/provider.ts
    - tests/process-manager.test.ts
    - tests/provider.test.ts

key-decisions:
  - "Used --effort levels instead of --thinking-budget tokens (CLI does not support --thinking-budget)"
  - "Opus detection via model.id.includes('opus') for forward-compatibility"
  - "Custom thinkingBudgets trigger console.warn, not error (CLI only supports effort levels)"
  - "Replaced local StreamOptions with SimpleStreamOptions & { cwd?: string } for type compatibility"

patterns-established:
  - "Effort mapping tables: two Record<ThinkingLevel, CliEffortLevel> maps for standard vs Opus models"
  - "Conditional CLI flag appending: only add --effort when effort value is computed (not undefined)"

requirements-completed: [CONF-02]

# Metrics
duration: 4min
completed: 2026-03-14
---

# Phase 3 Plan 1: Thinking Effort Mapping Summary

**Configurable thinking effort mapping from pi ThinkingLevel to CLI --effort flags, with Opus models getting elevated effort levels**

## Performance

- **Duration:** 4 min
- **Started:** 2026-03-14T17:41:33Z
- **Completed:** 2026-03-14T17:45:36Z
- **Tasks:** 2
- **Files modified:** 6

## Accomplishments
- Created thinking-config.ts module with dual effort mapping tables (standard and Opus) and isOpusModel detection
- Wired effort computation through provider.ts (SimpleStreamOptions -> mapThinkingEffort -> spawnClaude) with --effort flag appending in process-manager.ts
- Full TDD cycle: 22 thinking-config unit tests + 6 process-manager effort tests + 5 provider wiring tests = 33 new tests, all 187 total pass

## Task Commits

Each task was committed atomically (TDD: RED -> GREEN):

1. **Task 1: Create thinking-config module** (TDD)
   - `7d37495` (test: failing tests for effort mapping)
   - `6cb1dc9` (feat: implement thinking-config module)
2. **Task 2: Wire effort through process-manager, provider, index.ts** (TDD)
   - `206e3b0` (test: failing tests for effort flag wiring)
   - `771491f` (feat: wire effort through process-manager, provider)

## Files Created/Modified
- `src/thinking-config.ts` - Effort mapping tables, isOpusModel(), mapThinkingEffort()
- `tests/thinking-config.test.ts` - 22 tests covering all ThinkingLevel values, Opus elevation, xhigh downgrade, undefined reasoning, thinkingBudgets warning
- `src/process-manager.ts` - Added optional effort to spawnClaude options, --effort flag appending
- `src/provider.ts` - Imported SimpleStreamOptions and mapThinkingEffort, computes effort before spawning
- `tests/process-manager.test.ts` - 6 new tests for effort flag present/absent/backward-compatible
- `tests/provider.test.ts` - 5 new tests for reasoning -> effort -> spawnClaude wiring

## Decisions Made
- Used `--effort` levels instead of `--thinking-budget` tokens per research findings (CLI does not have --thinking-budget flag)
- Replaced local `StreamOptions` interface in provider.ts with `SimpleStreamOptions & { cwd?: string }` to stay type-compatible with pi's callback signature
- No changes needed in index.ts -- the existing `(model, context, options) => streamViaCli(model, context, options)` passthrough is already type-compatible since pi passes SimpleStreamOptions
- Used `CliEffortLevel` string type (not enum) for simplicity and direct string comparison

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Effort mapping complete and tested, ready for Plan 03-02 verification of STRM-04/STRM-05
- Full test suite green (187 tests), tsc --noEmit clean
- index.ts streamSimple passes SimpleStreamOptions through transparently

## Self-Check: PASSED

All 7 files verified present. All 4 task commits verified in git log.

---
*Phase: 03-extended-thinking-and-usage*
*Completed: 2026-03-14*
