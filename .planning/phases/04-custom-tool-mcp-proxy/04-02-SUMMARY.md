---
phase: 04-custom-tool-mcp-proxy
plan: 02
subsystem: tool-mapping
tags: [mcp, tool-mapping, prompt-builder, custom-tools, prefix-stripping]

# Dependency graph
requires:
  - phase: 04-custom-tool-mcp-proxy (plan 01)
    provides: MCP server infrastructure, break-early engine, mcp-config.ts, mcpConfigPath in provider/process-manager
provides:
  - MCP prefix stripping in mapClaudeToolNameToPi (mcp__custom-tools__deploy -> deploy)
  - isCustomToolName helper for distinguishing built-in vs custom tools
  - Custom tool history replay with MCP prefix in prompt-builder
  - Lazy MCP config generation in extension entry point (ensureMcpConfig)
  - mcpConfigPath wiring from index.ts through streamViaCli to spawnClaude
affects: [05-image-history-uat]

# Tech tracking
tech-stack:
  added: []
  patterns: [lazy-initialization, warn-dont-block, prefix-stripping, bidirectional-tool-name-mapping]

key-files:
  created: []
  modified:
    - src/tool-mapping.ts
    - src/prompt-builder.ts
    - index.ts
    - tests/tool-mapping.test.ts
    - tests/prompt-builder.test.ts
    - tests/event-bridge.test.ts

key-decisions:
  - "Lazy MCP config via ensureMcpConfig pattern — defers pi.getAllTools() to first request since it fails at extension load time"
  - "BUILT_IN_PI_NAMES derived from TOOL_MAPPINGS for consistent O(1) lookup"
  - "Custom tool args skip translatePiArgsToClaude in prompt-builder (passthrough, no renames)"
  - "Updated existing 'passes unknown tool names through unchanged' test to reflect new MCP-prefixing behavior"

patterns-established:
  - "Lazy init pattern: ensureMcpConfig caches result after first call, warn-don't-block on failure"
  - "Bidirectional custom tool name mapping: Claude uses mcp__custom-tools__ prefix, pi uses bare name"

requirements-completed: [MCP-02, CONF-01]

# Metrics
duration: 19min
completed: 2026-03-15
---

# Phase 4 Plan 2: Tool Name Mapping and Entry Point Wiring Summary

**MCP prefix stripping for bidirectional custom tool name translation, history replay awareness, and lazy config generation in extension entry point**

## Performance

- **Duration:** 19 min
- **Started:** 2026-03-15T02:52:40Z
- **Completed:** 2026-03-15T03:12:34Z
- **Tasks:** 2
- **Files modified:** 6

## Accomplishments
- MCP prefix stripping in mapClaudeToolNameToPi: `mcp__custom-tools__deploy` becomes `deploy` for pi, while other MCP server prefixes pass through unchanged
- Custom tool history replay in prompt-builder: custom tool calls and results use `mcp__custom-tools__` prefix so Claude recognizes them on replay
- Lazy MCP config generation in entry point: defers `pi.getAllTools()` to first LLM request (fails at load time), caches config path, warn-don't-block on failure
- Event bridge automatically benefits from prefix stripping via existing `mapClaudeToolNameToPi` call path (no event-bridge code changes needed)
- Full test coverage: 222 tests pass across 9 test files with zero regressions

## Task Commits

Each task was committed atomically:

1. **Task 1: MCP prefix stripping, custom tool history replay, event bridge tests**
   - `552bd87` (test) - TDD RED: failing tests for MCP prefix stripping and custom tool history replay
   - `28de1bc` (feat) - TDD GREEN: implement MCP prefix stripping and custom tool history replay
2. **Task 2: Wire MCP config generation into extension entry point** - `9c2a5c1` (feat)

## Files Created/Modified
- `src/tool-mapping.ts` - Added CUSTOM_TOOLS_MCP_PREFIX, BUILT_IN_PI_NAMES, isCustomToolName, MCP prefix stripping in mapClaudeToolNameToPi
- `src/prompt-builder.ts` - Custom tool history replay: MCP prefix for toolCall and toolResult, arg passthrough for custom tools
- `index.ts` - Lazy ensureMcpConfig pattern, mcpConfigPath wiring to streamViaCli
- `tests/tool-mapping.test.ts` - MCP prefix stripping tests, isCustomToolName tests, MCP arg passthrough test
- `tests/prompt-builder.test.ts` - Custom tool history replay tests (5 new), updated existing unknown-tool test
- `tests/event-bridge.test.ts` - MCP prefix stripping through event bridge path (2 new tests)

## Decisions Made
- **Lazy MCP config via ensureMcpConfig:** pi.getAllTools() fails during extension loading (discovered in 04-01). Solution: defer to first LLM request with single-call caching and warn-don't-block error handling.
- **BUILT_IN_PI_NAMES from TOOL_MAPPINGS:** Single source of truth for built-in tool names, avoids duplication with mcp-config.ts's BUILT_IN_TOOL_NAMES (both derive from same conceptual set, but tool-mapping uses TOOL_MAPPINGS as canonical source).
- **Custom tool args passthrough:** Custom tools have no argument renames (unlike built-ins), so prompt-builder skips translatePiArgsToClaude for custom tools.
- **Updated existing test assertion:** The "passes unknown tool names through unchanged" test now correctly asserts MCP-prefixed output for custom tool names, reflecting the new behavior.

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Phase 4 (Custom Tool MCP Proxy) is now complete: all MCP infrastructure (plan 01) and tool name mapping + entry point wiring (plan 02) are in place
- The full chain is wired: index.ts -> ensureMcpConfig -> getCustomToolDefs/writeMcpConfig -> streamViaCli -> spawnClaude with mcpConfigPath
- Custom tool names translate bidirectionally: Claude's `mcp__custom-tools__X` <-> pi's `X`
- Ready for Phase 5 (Image History and UAT)

## Self-Check: PASSED

- All 6 modified files exist on disk
- All 3 task commits verified (552bd87, 28de1bc, 9c2a5c1)
- SUMMARY.md exists at expected path
- 222 tests pass across 9 test files

---
*Phase: 04-custom-tool-mcp-proxy*
*Completed: 2026-03-15*
