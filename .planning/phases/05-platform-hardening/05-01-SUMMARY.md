---
phase: 05-platform-hardening
plan: 01
subsystem: infra
tags: [subprocess, process-management, SIGKILL, timeout, error-handling, cross-platform]

# Dependency graph
requires:
  - phase: 01-core-subprocess-bridge
    provides: process-manager.ts (spawnClaude, cleanupProcess, captureStderr)
  - phase: 04-custom-tool-mcp-proxy
    provides: provider.ts break-early pattern, broken flag, event bridge done/end async flow
provides:
  - forceKillProcess with killed/exitCode guards (cross-platform SIGKILL)
  - registerProcess / killAllProcesses global process registry
  - cleanupProcess with 500ms grace period
  - Inactivity timeout (180s) with reset on each stdout line
  - endStreamWithError helper with streamEnded + broken guards
  - proc.on("close") handler surfacing stderr + exit code
  - Abort handler using SIGKILL instead of SIGTERM
  - process.on("exit", killAllProcesses) teardown hook
affects: [05-02-PLAN, provider.ts consumers]

# Tech tracking
tech-stack:
  added: []
  patterns: [streamEnded guard, endStreamWithError helper, inactivity timeout reset, process registry Set]

key-files:
  created: []
  modified: [src/process-manager.ts, src/provider.ts, index.ts, tests/process-manager.test.ts, tests/provider.test.ts]

key-decisions:
  - "proc.kill('SIGKILL') is cross-platform safe -- Node.js treats all kill signals as forceful termination on Windows"
  - "180-second inactivity timeout -- covers documented 3+ minute thinking gaps with safety margin"
  - "endStreamWithError checks both streamEnded AND broken to prevent double errors and post-break-early errors"
  - "Inactivity timer reset placed AFTER broken guard in line handler per plan coordination note"

patterns-established:
  - "streamEnded guard: first error path wins, subsequent are no-ops"
  - "endStreamWithError helper: centralized error+end with guard checks"
  - "Process registry pattern: Set<ChildProcess> with auto-remove on exit"
  - "Inactivity timeout pattern: setTimeout/clearTimeout reset on each readline line event"

requirements-completed: [PROC-02, PROC-03, PROC-04]

# Metrics
duration: 5min
completed: 2026-03-15
---

# Phase 5 Plan 1: Subprocess Lifecycle Hardening Summary

**Force-kill via SIGKILL with 500ms grace, 180s inactivity timeout, subprocess crash error surfacing, streamEnded double-error guard, and global process registry with teardown cleanup**

## Performance

- **Duration:** 5 min
- **Started:** 2026-03-15T05:47:00Z
- **Completed:** 2026-03-15T05:52:34Z
- **Tasks:** 2
- **Files modified:** 5

## Accomplishments
- Hardened subprocess lifecycle: forceKillProcess, process registry, 500ms cleanupProcess grace period
- Inactivity timeout (180s) detects stuck/crashed subprocesses and surfaces meaningful errors
- Subprocess crash handling surfaces stderr content and exit code as error events
- streamEnded guard eliminates double error events and double stream.end() calls
- Abort handler changed from SIGTERM to SIGKILL for reliable cross-platform cancellation
- Global process registry with teardown hook prevents orphaned claude processes

## Task Commits

Each task was committed atomically (TDD: RED then GREEN):

1. **Task 1: forceKillProcess, process registry, cleanupProcess 500ms** - RED `b20c0dc` / GREEN `2071df9`
2. **Task 2: Inactivity timeout, close handler, streamEnded guard, abort fix, index.ts registry** - RED `843147f` / GREEN `1103169`

## Files Created/Modified
- `src/process-manager.ts` - Added forceKillProcess, registerProcess, killAllProcesses; updated cleanupProcess to 500ms
- `src/provider.ts` - Added inactivity timeout, proc.on("close") handler, streamEnded guard, endStreamWithError helper, abort SIGKILL, registerProcess integration
- `index.ts` - Added process.on("exit", killAllProcesses) teardown hook
- `tests/process-manager.test.ts` - Added 9 new tests for forceKillProcess, process registry; updated cleanupProcess grace period tests
- `tests/provider.test.ts` - Added 8 new tests for subprocess error handling, inactivity timeout, abort SIGKILL; updated grace period in existing tests

## Decisions Made
- Used `proc.kill("SIGKILL")` everywhere (cross-platform safe) rather than platform-specific taskkill -- Node.js handles Windows abstraction
- Set inactivity timeout to 180 seconds (3 minutes) based on documented thinking gap worst cases
- endStreamWithError checks both `streamEnded` AND `broken` flags per plan coordination notes
- Inactivity timer reset placed AFTER `broken` guard in line handler (per plan's updated coordination note: if broken, we are about to close, don't reset timer)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Mock proc missing exitCode property**
- **Found during:** Task 2 (GREEN phase)
- **Issue:** Provider test mock's EventEmitter proc had no `exitCode` property (undefined), causing `forceKillProcess` guard `proc.exitCode !== null` to return true for `undefined !== null`, making all SIGKILL calls no-op
- **Fix:** Added `(proc as any).exitCode = null` to the cross-spawn mock in provider.test.ts
- **Files modified:** tests/provider.test.ts
- **Verification:** All 30 provider tests pass
- **Committed in:** `1103169` (Task 2 GREEN commit)

---

**Total deviations:** 1 auto-fixed (1 bug)
**Impact on plan:** Essential fix for mock to match real ChildProcess API. No scope creep.

## Issues Encountered
None -- plan executed smoothly after the mock fix.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Subprocess lifecycle fully hardened with force-kill, timeout, error surfacing, and registry
- PROC-02 (cross-spawn) verified as already satisfied
- PROC-03 (force-kill after result) complete with 500ms grace and SIGKILL
- PROC-04 (error surfacing) complete with stderr, exit code, timeout, and streamEnded guard
- Ready for Phase 5 Plan 2 (image history replay / HIST-02)

## Self-Check: PASSED

All 6 files verified present. All 4 commit hashes verified in git log.

---
*Phase: 05-platform-hardening*
*Completed: 2026-03-15*
