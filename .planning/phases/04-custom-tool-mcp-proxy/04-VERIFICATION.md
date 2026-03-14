---
phase: 04-custom-tool-mcp-proxy
verified: 2026-03-14T22:30:00Z
status: passed
score: 14/14 must-haves verified
re_verification:
  previous_status: passed
  previous_score: 7/7
  note: "Previous verification was written against a now-reverted implementation (referenced src/mcp-server.ts which does not exist). This re-verification targets the current break-early architecture implementation."
  gaps_closed:
    - "All 14 must-haves from 04-01-PLAN and 04-02-PLAN verified against actual codebase"
  gaps_remaining: []
  regressions: []
---

# Phase 4: Custom Tool MCP Proxy Verification Report

**Phase Goal:** Pi's custom tools (non-built-in) are available to Claude via MCP, enabling Claude to propose custom tool calls
**Verified:** 2026-03-14T22:30:00Z
**Status:** passed
**Re-verification:** Yes — previous VERIFICATION.md was written against a reverted implementation; this verification targets the current break-early architecture

## Note on Previous Verification

The previous VERIFICATION.md (status: passed, score 7/7) referenced `src/mcp-server.ts` — a file that does not exist in the codebase. The git log confirms the original MCP proxy was reverted (`758c827 revert(04): remove MCP proxy — architecture fundamentally flawed`) and replaced with a break-early architecture. The previous verification described an implementation that was never committed in its referenced form. This document verifies the actual current implementation.

---

## Goal Achievement

### Observable Truths — Plan 04-01 (MCP-01, CONF-01)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Custom pi tools are exposed to Claude via a schema-only stdio MCP server registered via `--mcp-config` | VERIFIED | `src/mcp-schema-server.cjs` exists (36 lines), handles `initialize` + `tools/list` via raw JSON-RPC; `src/mcp-config.ts` exports `writeMcpConfig` which writes config pointing to this server; `index.ts` calls `writeMcpConfig` via `ensureMcpConfig`; `spawnClaude` receives `--mcp-config` flag |
| 2 | `spawnClaude` uses `--permission-prompt-tool stdio` (not `--permission-mode dontAsk`) and passes `--mcp-config` | VERIFIED | `src/process-manager.ts` line 34: `"--permission-prompt-tool", "stdio"` is present; no `--permission-mode` or `dontAsk` anywhere in source; lines 45-47: `if (options?.mcpConfigPath) { args.push("--mcp-config", options.mcpConfigPath); }` |
| 3 | Existing user MCP configs load automatically (no `--strict-mcp-config`) | VERIFIED | `--strict-mcp-config` confirmed absent from all source files; process-manager tests assert this explicitly |
| 4 | At `message_stop`, subprocess is killed early if any built-in or `mcp__custom-tools__*` tool was seen | VERIFIED | `src/provider.ts` lines 131-148: tracks `sawBuiltInOrCustomTool`; at `message_stop` with flag true: sets `broken = true`, calls `proc.kill("SIGKILL")`, calls `rl.close()`; 4 break-early provider tests pass including built-in and custom-tools MCP cases |
| 5 | Control handler allows ALL control_requests (only reached when no break-early) | VERIFIED | `src/control-handler.ts` lines 61-68: unconditionally builds `{ behavior: "allow" }` response for any valid request; 6 allow tests pass (built-in, MCP, custom-tools, unknown tools); no deny path exists |
| 6 | If MCP setup fails, provider still registers and built-in tools still work | VERIFIED | `index.ts` lines 29-38: `ensureMcpConfig` has try/catch; `console.warn` on failure; `mcpConfigPath` remains undefined; `streamViaCli` is called without `mcpConfigPath` — subprocess spawns without `--mcp-config`; provider registration test confirms warn-don't-block behavior |
| 7 | After break-early fires, the stream emits a `done` event (from event bridge) before ending | VERIFIED | `src/provider.ts` line 128: `bridge.handleEvent(msg.event)` is called BEFORE the break-early check at line 140; `message_stop` event is processed by the bridge (which pushes `done`) before `broken` is set and `rl.close()` is called; provider break-early test asserts `done` event received |
| 8 | Buffered readline lines after `rl.close()` are guarded by a `broken` flag and do not fire | VERIFIED | `src/provider.ts` line 122: `if (broken) return;` is the first statement in `rl.on("line", ...)` callback; `broken = true` is set BEFORE `rl.close()` at line 141-142 |

### Observable Truths — Plan 04-02 (MCP-02, CONF-01)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 9 | When Claude proposes `mcp__custom-tools__<name>`, pi receives it as just `<name>` | VERIFIED | `src/tool-mapping.ts` lines 79-82: `if (claudeName.startsWith(CUSTOM_TOOLS_MCP_PREFIX)) { return claudeName.slice(CUSTOM_TOOLS_MCP_PREFIX.length); }`; 5 MCP prefix stripping tests pass; 2 event-bridge integration tests confirm end-to-end |
| 10 | Built-in tool name mapping still works unchanged (Read->read, Glob->find, etc.) | VERIFIED | Existing built-in lookup unchanged; prefix stripping only fires on `mcp__custom-tools__` prefix; `mapClaudeToolNameToPi("Read")` returns `"read"` confirmed by test |
| 11 | Custom tool results in history replay use `mcp__custom-tools__` prefix so Claude recognizes them | VERIFIED | `src/prompt-builder.ts` lines 38-41: `isCustomToolName` guard applies `CUSTOM_TOOLS_MCP_PREFIX + message.toolName` for toolResult; lines 115-118: same guard in `contentToText` for toolCall blocks; 5 prompt-builder tests confirm both paths |
| 12 | Extension entry point generates MCP config with custom tool schemas and passes config path to provider | VERIFIED | `index.ts` lines 26-39: `ensureMcpConfig(pi)` lazy-generates on first call; calls `getCustomToolDefs(pi)` + `writeMcpConfig(toolDefs)`; lines 62-64: `const configPath = ensureMcpConfig(pi)` → `streamViaCli(model, context, { ...options, mcpConfigPath: configPath })` |
| 13 | MCP setup failure does not block provider registration (warn-don't-block) | VERIFIED | `index.ts` lines 35-38: catch block calls `console.warn` and returns `mcpConfigPath` as undefined; outer try/catch at lines 67-69 keeps provider registration alive even if validation fails |
| 14 | Event bridge strips `mcp__custom-tools__` prefix via `mapClaudeToolNameToPi` (verified by automated test) | VERIFIED | `tests/event-bridge.test.ts` lines 860-899: `describe("MCP prefix stripping via mapClaudeToolNameToPi")` with 2 tests confirming `toolcall_start` receives name `"foo"` (not `"mcp__custom-tools__foo"`) |

**Score:** 14/14 truths verified

---

### Required Artifacts

#### Plan 04-01 Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `src/mcp-schema-server.cjs` | Schema-only stdio MCP server (~30 lines) reading tool schemas from temp file | VERIFIED | 36 lines, CommonJS, handles `initialize` + `tools/list` via raw JSON-RPC; no `console.log`; no SDK dependency; reads schemas from `process.argv[2]` |
| `src/mcp-config.ts` | Custom tool discovery and MCP config file generation | VERIFIED | Exports `getCustomToolDefs` (filters 6 built-ins) and `writeMcpConfig` (writes 2 temp files, returns config path); 77 lines, substantive |
| `src/process-manager.ts` | `spawnClaude` with `--permission-prompt-tool stdio` and `--mcp-config` option | VERIFIED | Line 34: `--permission-prompt-tool stdio` present; `mcpConfigPath?: string` in options type; lines 45-47: conditional `--mcp-config` append |
| `src/control-handler.ts` | Simplified control handler that allows all control_requests | VERIFIED | 73 lines; `behavior: "allow"` unconditional (no deny path); guard for malformed messages; backward-compat exports retained |
| `src/provider.ts` | Break-early logic at `message_stop`, `mcpConfigPath` passthrough, `broken` guard | VERIFIED | Line 31: `mcpConfigPath?: string` in `StreamViaCLiOptions`; lines 70-71: passed to `spawnClaude`; lines 96-148: `sawBuiltInOrCustomTool`, `broken` tracking, kill + `rl.close()` at `message_stop` |

#### Plan 04-02 Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `src/tool-mapping.ts` | `CUSTOM_TOOLS_MCP_PREFIX` constant, MCP prefix stripping in `mapClaudeToolNameToPi`, `isCustomToolName` helper | VERIFIED | Line 32: `export const CUSTOM_TOOLS_MCP_PREFIX = "mcp__custom-tools__"`; line 41: `export function isCustomToolName`; lines 78-85: prefix stripping in `mapClaudeToolNameToPi` |
| `src/prompt-builder.ts` | Custom tool names prefixed with `mcp__custom-tools__` during history replay | VERIFIED | Line 13: imports `isCustomToolName, CUSTOM_TOOLS_MCP_PREFIX`; lines 38-41: toolResult path; lines 114-118: toolCall path; both correctly conditional on `isCustomToolName` |
| `index.ts` | Lazy MCP config generation, `mcpConfigPath` passed to `streamViaCli` | VERIFIED | `ensureMcpConfig` pattern (lines 26-39); `mcpConfigPath` flows into `streamViaCli` at line 64; imports from `./src/mcp-config.js` |
| `tests/event-bridge.test.ts` | Test confirming `mcp__custom-tools__foo` is stripped to `foo` by the event bridge | VERIFIED | Lines 860-899: `describe("MCP prefix stripping via mapClaudeToolNameToPi")` with 2 integration tests; both pass |

---

### Key Link Verification

#### Plan 04-01 Key Links

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `src/mcp-config.ts` | `src/mcp-schema-server.cjs` | `writeMcpConfig` config JSON references `.cjs` file path | WIRED | Line 59: `const serverPath = join(__dirname, "mcp-schema-server.cjs")`; used as `args[0]` in config JSON at line 65 |
| `src/process-manager.ts` | Claude CLI | `--mcp-config` flag in spawn args | WIRED | Lines 45-47: `args.push("--mcp-config", options.mcpConfigPath)` inside `mcpConfigPath` guard |
| `src/provider.ts` | `src/process-manager.ts` | passes `mcpConfigPath` to `spawnClaude` options | WIRED | Line 70-71: `mcpConfigPath: options?.mcpConfigPath` in `spawnClaude` call |

#### Plan 04-02 Key Links

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `src/tool-mapping.ts` | `src/event-bridge.ts` | `mapClaudeToolNameToPi` strips `mcp__custom-tools__` prefix — event bridge calls this automatically | WIRED | Event bridge calls `mapClaudeToolNameToPi` in `handleContentBlockStart`; prefix stripping fires automatically; confirmed by 2 event-bridge integration tests |
| `src/prompt-builder.ts` | `src/tool-mapping.ts` | `isCustomToolName` + `CUSTOM_TOOLS_MCP_PREFIX` for conditional prefix in history replay | WIRED | Line 13: both imported; used in `contentToText` (toolCall) and `buildPrompt` (toolResult) |
| `index.ts` | `src/mcp-config.ts` | `getCustomToolDefs` for discovery, `writeMcpConfig` for config generation | WIRED | Line 12: `import { getCustomToolDefs, writeMcpConfig } from "./src/mcp-config.js"`; both called in `ensureMcpConfig` |
| `index.ts` | `src/provider.ts` | passes `mcpConfigPath` to `streamViaCli` options | WIRED | Line 64: `return streamViaCli(model, context, { ...options, mcpConfigPath: configPath })` |

---

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| MCP-01 | 04-01 | Extension exposes custom pi tools (non-built-in) to Claude via a stdio MCP server, registered with the subprocess via `--mcp-config` | SATISFIED | `src/mcp-schema-server.cjs` is the stdio server; `src/mcp-config.ts` generates the config; `index.ts` calls both lazily; `spawnClaude` passes `--mcp-config`; 9 mcp-config tests pass |
| MCP-02 | 04-02 | Extension strips the `mcp__custom-tools__` prefix from tool call names when mapping MCP tool proposals back to pi tool names | SATISFIED | `mapClaudeToolNameToPi` strips prefix; event bridge auto-benefits; 5 tool-mapping prefix tests + 2 event-bridge integration tests pass |
| CONF-01 | 04-01, 04-02 | Extension defaults `strictMcpConfig` to `false`, allowing existing `.mcp.json` and `~/.claude.json` MCP server configurations to load automatically | SATISFIED | `--strict-mcp-config` confirmed absent from all source; `--permission-prompt-tool stdio` used instead of `dontAsk`; process-manager tests assert absence; user MCP tools allowed via control_request path (don't-break path) |

**Orphaned requirements check:** REQUIREMENTS.md maps MCP-01, MCP-02, CONF-01 to Phase 4. All three are claimed by plan frontmatter (MCP-01 and CONF-01 in 04-01; MCP-02 and CONF-01 in 04-02) and verified above. No orphaned requirements.

---

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `src/prompt-builder.ts` | 125 | Comment uses word "placeholder" in `// Unknown block types are represented as a placeholder` | Info | Not a stub — the `return \`[${block.type}]\`` on line 126 is a legitimate fallback for unknown block types, not unimplemented code |

No blocker or warning anti-patterns found. Checked all phase-modified files for:
- `console.log` in `src/mcp-schema-server.cjs` — NONE (only `console.error` in `src/control-handler.ts`)
- `--strict-mcp-config` anywhere functional — NONE
- `--permission-mode dontAsk` — NONE
- Empty implementations, TODO/FIXME, return null/return [] stubs — NONE
- Deny logic remaining in `control-handler.ts` — NONE (fully replaced with allow-all)

---

### Test Suite Results

**222 tests, 9 test files, 0 failures.**

Key test coverage for Phase 4:
- `tests/mcp-config.test.ts`: 9 tests — `getCustomToolDefs` (4) + `writeMcpConfig` (5)
- `tests/process-manager.test.ts`: CLI flag assertions — `--permission-prompt-tool stdio` present, `--permission-mode dontAsk` absent, `--strict-mcp-config` absent, `--mcp-config` conditional
- `tests/control-handler.test.ts`: 6+ allow-all tests (built-in, MCP, custom-tools, unknown), malformed guard
- `tests/provider.test.ts`: `mcpConfigPath` passthrough (1 test) + break-early logic (4 tests: built-in kill + done event, custom-tools kill, no-break for text-only, no-break for user-MCP-only)
- `tests/tool-mapping.test.ts`: 5 MCP prefix stripping tests + 2 `isCustomToolName` tests
- `tests/prompt-builder.test.ts`: 5 custom tool history replay tests
- `tests/event-bridge.test.ts`: 2 MCP prefix stripping integration tests

---

### Human Verification Required

#### 1. End-to-end custom tool exposure to Claude

**Test:** Load the extension in a live pi session with a custom tool registered. Inspect the Claude subprocess invocation to confirm `--mcp-config` flag is present and the temp JSON file exists with correct content.
**Expected:** A `pi-claude-mcp-config-<pid>.json` file in `os.tmpdir()` contains a valid `mcpServers.custom-tools` entry pointing to `mcp-schema-server.cjs`, with custom tool schemas serialized in a sibling `pi-claude-mcp-schemas-<pid>.json` file.
**Why human:** Requires a live pi instance with an actual custom tool extension loaded; `pi.getAllTools()` must return non-empty results.

#### 2. Claude sees custom tools in its tool list

**Test:** In a live session with custom tools registered, ask Claude what tools it has access to.
**Expected:** Claude lists the custom tool names (with `mcp__custom-tools__` prefix as the server exposes them), and is able to propose calling them.
**Why human:** Requires the full Claude CLI subprocess to start, read the MCP config, and spawn the MCP server — integration behavior beyond unit test scope.

#### 3. Break-early round-trip with custom tool call

**Test:** In a live session, ask Claude to perform an action requiring a custom tool. Observe that the subprocess is killed at `message_stop` and pi receives the tool call with the pi tool name (no MCP prefix).
**Expected:** Pi's tool handler receives `name: "myCustomTool"` (not `name: "mcp__custom-tools__myCustomTool"`). The subprocess is killed before Claude auto-executes it.
**Why human:** Requires live MCP server, live Claude subprocess, real tool call flow, and observable subprocess PID lifecycle.

#### 4. User MCP tools work in the don't-break path

**Test:** Configure a user-side MCP tool in `.mcp.json`. Ask Claude to use it. Verify pi does NOT kill the subprocess at `message_stop` for user MCP tools.
**Expected:** Claude executes the user MCP tool via its own MCP server (the control_request is allowed); pi receives the result in the conversation.
**Why human:** Requires a live user-side MCP server, real control_request/control_response exchange, and observable stream continuation past `message_stop`.

---

### Gaps Summary

No gaps. All 14 observable truths verified, all 9 artifacts are substantive and correctly wired, all 7 key links confirmed, all 3 requirements satisfied.

The test suite passes with 222 tests and 0 failures. The break-early architecture is fully implemented: `--permission-prompt-tool stdio` enables control_request for MCP tools, the subprocess is killed at `message_stop` before Claude can auto-execute built-in or custom tools, and the `broken` guard prevents race conditions with buffered readline lines.

The previous VERIFICATION.md (claiming a different implementation) should be considered superseded by this document.

---

_Verified: 2026-03-14T22:30:00Z_
_Verifier: Claude (gsd-verifier)_
