---
phase: 04-custom-tool-mcp-proxy
plan: 01
subsystem: mcp
tags: [mcp, json-rpc, stdio, break-early, subprocess, cli-flags]

# Dependency graph
requires:
  - phase: 01-core-subprocess-bridge
    provides: process-manager spawnClaude, provider streamViaCli, event bridge
  - phase: 02-tool-handling
    provides: control-handler, tool-mapping TOOL_MAPPINGS
provides:
  - Schema-only stdio MCP server (mcp-schema-server.cjs) for Claude tool discovery
  - Config generation module (mcp-config.ts) with getCustomToolDefs and writeMcpConfig
  - Break-early subprocess control at message_stop in provider.ts
  - Simplified allow-all control handler for user MCP passthrough
  - --permission-prompt-tool stdio flag (replaces --permission-mode dontAsk)
  - --mcp-config flag support in spawnClaude
affects: [04-02, 05-platform-hardening]

# Tech tracking
tech-stack:
  added: []
  patterns: [break-early subprocess kill, schema-only MCP, raw JSON-RPC stdio]

key-files:
  created:
    - src/mcp-schema-server.cjs
    - src/mcp-config.ts
    - tests/mcp-config.test.ts
  modified:
    - src/process-manager.ts
    - src/control-handler.ts
    - src/provider.ts
    - tests/process-manager.test.ts
    - tests/control-handler.test.ts
    - tests/provider.test.ts

key-decisions:
  - "Raw JSON-RPC stdio over mcp-schema-server.cjs (no SDK dependency, ~35 lines)"
  - "Break-early kills subprocess with SIGKILL at message_stop before CLI auto-executes tools"
  - "broken flag guards against buffered readline lines after rl.close()"
  - "Control handler allows ALL control_requests (only reached when no break-early)"
  - "fileURLToPath(import.meta.url) for resolving .cjs sibling in ESM/TS module system"
  - "Path separator normalization in test assertions for Windows compatibility"

patterns-established:
  - "Break-early pattern: at message_stop, check sawBuiltInOrCustomTool, kill proc, set broken guard"
  - "Schema-only MCP: server only implements initialize + tools/list, tools/call never reached"
  - "Config temp files: pid-suffixed JSON files in tmpdir for MCP schema and config"

requirements-completed: [MCP-01, CONF-01]

# Metrics
duration: 14min
completed: 2026-03-15
---

# Phase 4 Plan 1: MCP Server Infrastructure Summary

**Schema-only stdio MCP server with break-early subprocess control, config generation for custom tool discovery, and simplified allow-all control handler**

## Performance

- **Duration:** 14 min
- **Started:** 2026-03-15T02:28:06Z
- **Completed:** 2026-03-15T02:42:09Z
- **Tasks:** 2
- **Files modified:** 9

## Accomplishments
- Created minimal raw JSON-RPC stdio MCP server (~35 lines) that implements initialize + tools/list for Claude tool schema discovery
- Built config generation module that filters built-in tools, writes schemas to temp files, and generates MCP config pointing to schema server
- Implemented break-early pattern: provider kills subprocess at message_stop when built-in or custom-tools MCP tool_use blocks are seen, preventing CLI from auto-executing tools
- Simplified control handler to allow ALL control_requests (only reached in don't-break path for user MCPs)
- Changed CLI flags from --permission-mode dontAsk to --permission-prompt-tool stdio, enabling control_request for MCP tools
- Added --mcp-config flag support in spawnClaude for registering MCP server with subprocess

## Task Commits

Each task was committed atomically (TDD: test then feat):

1. **Task 1: MCP schema server, config generation, process-manager flags**
   - `5ef9a9b` (test: failing tests)
   - `2d7bb5c` (feat: implementation)
2. **Task 2: Break-early logic in provider, simplified control handler**
   - `490bafa` (test: failing tests)
   - `2e28d58` (feat: implementation)

## Files Created/Modified
- `src/mcp-schema-server.cjs` - Minimal stdio JSON-RPC MCP server (initialize + tools/list)
- `src/mcp-config.ts` - Custom tool discovery (getCustomToolDefs) and MCP config generation (writeMcpConfig)
- `src/process-manager.ts` - --permission-prompt-tool stdio, --mcp-config flag, mcpConfigPath option
- `src/control-handler.ts` - Simplified to allow ALL control_requests
- `src/provider.ts` - Break-early logic, broken guard, mcpConfigPath passthrough, tool_use tracking
- `tests/mcp-config.test.ts` - 9 tests for getCustomToolDefs and writeMcpConfig
- `tests/process-manager.test.ts` - 8 new tests for CLI flags and mcp-config flag
- `tests/control-handler.test.ts` - Rewritten for allow-all behavior (16 tests)
- `tests/provider.test.ts` - 5 new tests for break-early and mcpConfigPath

## Decisions Made
- Used raw JSON-RPC stdio instead of MCP SDK (no dependency needed, server is ~35 lines)
- Break-early uses SIGKILL (immediate) rather than SIGTERM (graceful) since subprocess state is not needed
- broken flag set BEFORE rl.close() to prevent race condition with buffered readline lines
- Control handler allows everything -- deny logic removed since break-early is the deny mechanism
- fileURLToPath(import.meta.url) for ESM-compatible __dirname resolution to find .cjs sibling

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- MCP server infrastructure ready for plan 04-02 (tool call mapping, MCP prefix stripping, integration)
- All 206 tests pass (19 new tests added by this plan)
- Break-early pattern verified with both built-in tools and mcp__custom-tools__ prefixed tools

## Self-Check: PASSED

All 9 files verified present. All 4 task commits verified in git log.

---
*Phase: 04-custom-tool-mcp-proxy*
*Completed: 2026-03-15*
