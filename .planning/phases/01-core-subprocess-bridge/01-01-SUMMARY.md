---
phase: 01-core-subprocess-bridge
plan: 01
subsystem: foundation
tags: [typescript, vitest, pi-package, wire-protocol, prompt-builder]

# Dependency graph
requires: []
provides:
  - "Project scaffold with package.json, tsconfig.json, vitest.config.ts"
  - "Wire protocol types for all NDJSON message types (src/types.ts)"
  - "Prompt builder with history flattening and AGENTS.md system prompt (src/prompt-builder.ts)"
affects: [01-02, 01-03, tool-handling, extended-thinking, platform-hardening]

# Tech tracking
tech-stack:
  added: [typescript, vitest, cross-spawn]
  patterns: [pi-package-metadata, tdd-red-green, content-block-serialization]

key-files:
  created:
    - package.json
    - tsconfig.json
    - vitest.config.ts
    - src/types.ts
    - src/prompt-builder.ts
    - tests/prompt-builder.test.ts
  modified: []

key-decisions:
  - "Used any type for Context to avoid requiring @mariozechner/pi-ai at dev time"
  - "Separate content-to-text helpers for user, assistant, and tool result messages"
  - "AGENTS.md walk-up resolution with global fallback to ~/.pi/agent/AGENTS.md"

patterns-established:
  - "Content block serialization: text->text, thinking->thinking text, toolCall->historical non-executable string"
  - "Role labeling: USER:/ASSISTANT:/TOOL RESULT (historical name): prefix pattern"
  - "Pi sanitization: .pi->.claude path replacement in AGENTS.md content"

requirements-completed: [PROV-02, HIST-01]

# Metrics
duration: 4min
completed: 2026-03-14
---

# Phase 1 Plan 01: Project Scaffold and Prompt Builder Summary

**TypeScript project scaffold with wire protocol types and TDD-driven prompt builder that flattens pi conversation history into labeled USER/ASSISTANT/TOOL RESULT text blocks**

## Performance

- **Duration:** 4 min
- **Started:** 2026-03-14T03:30:49Z
- **Completed:** 2026-03-14T03:35:32Z
- **Tasks:** 2
- **Files modified:** 7

## Accomplishments
- Project skeleton established with pi-package metadata, peer deps, and dev tooling
- All NDJSON wire protocol types defined (ClaudeStreamEventMessage, ClaudeResultMessage, ClaudeSystemMessage, ClaudeControlRequest, ClaudeApiEvent, ClaudeUsage, TrackedContentBlock)
- Prompt builder implements history flattening with USER:/ASSISTANT:/TOOL RESULT: labels
- System prompt builder loads AGENTS.md with directory walk-up and sanitizes .pi references to .claude
- 14 unit tests covering all prompt builder behaviors

## Task Commits

Each task was committed atomically:

1. **Task 1: Scaffold project and define shared types** - `3b6b003` (feat)
2. **Task 2 RED: Failing prompt builder tests** - `d104e6f` (test)
3. **Task 2 GREEN: Implement prompt builder** - `fb97a8c` (feat)

## Files Created/Modified
- `package.json` - Project manifest with pi-package metadata, peer deps, scripts
- `tsconfig.json` - TypeScript config for ESNext/bundler with strict mode
- `vitest.config.ts` - Vitest test runner configuration with globals
- `.gitignore` - Excludes node_modules and dist
- `src/types.ts` - Wire protocol types for all NDJSON message types
- `src/prompt-builder.ts` - History flattening and system prompt builder
- `tests/prompt-builder.test.ts` - 14 unit tests for prompt builder

## Decisions Made
- Used `any` type for Context parameter to avoid requiring `@mariozechner/pi-ai` at dev time; pi provides the real type at runtime
- Created separate content-to-text helpers (userContentToText, contentToText, toolResultContentToText) for clarity and different handling per role
- AGENTS.md resolution walks up from cwd then falls back to `~/.pi/agent/AGENTS.md`

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Added .gitignore for node_modules**
- **Found during:** Task 1 (project scaffolding)
- **Issue:** No .gitignore existed; node_modules would be committed
- **Fix:** Created .gitignore excluding node_modules/ and dist/
- **Files modified:** .gitignore
- **Verification:** git status shows node_modules excluded
- **Committed in:** 3b6b003 (Task 1 commit)

---

**Total deviations:** 1 auto-fixed (1 blocking)
**Impact on plan:** Essential for correct git hygiene. No scope creep.

## Issues Encountered
- Test assertion for image skipping initially used `toContain("image")` which matched the word "image" in user text "Look at this image" -- fixed assertion to check for base64 data and [image] tag markers instead

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness
- Wire protocol types ready for stream parser and event bridge (Plan 01-02)
- Prompt builder ready for integration in provider handler (Plan 01-03)
- All dependencies installed and test infrastructure working

---
## Self-Check: PASSED

All 7 files verified present. All 3 commits verified in git log.

---
*Phase: 01-core-subprocess-bridge*
*Completed: 2026-03-14*
