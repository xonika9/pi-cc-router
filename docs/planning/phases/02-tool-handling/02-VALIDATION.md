---
phase: 2
slug: tool-handling
status: validated
nyquist_compliant: true
wave_0_complete: true
created: 2026-03-13
---

# Phase 2 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property               | Value                               |
| ---------------------- | ----------------------------------- |
| **Framework**          | vitest ^3.0.0                       |
| **Config file**        | vitest.config.ts                    |
| **Quick run command**  | `npx vitest run --reporter=verbose` |
| **Full suite command** | `npx vitest run --reporter=verbose` |
| **Estimated runtime**  | ~3 seconds                          |

---

## Sampling Rate

- **After every task commit:** Run `npx vitest run --reporter=verbose`
- **After every plan wave:** Run `npx vitest run --reporter=verbose`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 3 seconds

---

## Per-Task Verification Map

| Req ID  | Requirement                                            | Test Type | Automated Command                                                        | File Exists | Status   |
| ------- | ------------------------------------------------------ | --------- | ------------------------------------------------------------------------ | ----------- | -------- |
| STRM-03 | Control response messages on stdin for tool deny/allow | unit      | `npx vitest run tests/control-handler.test.ts`                           | ✅          | ✅ green |
| TOOL-01 | Deny all built-in tool execution via control protocol  | unit      | `npx vitest run tests/control-handler.test.ts -t "deny"`                 | ✅          | ✅ green |
| TOOL-02 | Allow mcp\_\_ prefixed tool execution                  | unit      | `npx vitest run tests/control-handler.test.ts -t "allows"`               | ✅          | ✅ green |
| TOOL-03 | Bidirectional tool name mapping (Claude <-> pi)        | unit      | `npx vitest run tests/tool-mapping.test.ts -t "mapClaudeToolNameToPi"`   | ✅          | ✅ green |
| TOOL-04 | Translate tool arguments between formats               | unit      | `npx vitest run tests/tool-mapping.test.ts -t "translateClaudeArgsToPi"` | ✅          | ✅ green |

_Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky_

---

## Wave 0 Requirements

- [x] `tests/tool-mapping.test.ts` — TOOL-03, TOOL-04 (tool name + arg mapping)
- [x] `tests/control-handler.test.ts` — STRM-03, TOOL-01, TOOL-02 (control protocol)
- [x] Tool_use content block streaming tests in `tests/event-bridge.test.ts`

_Existing test infrastructure from Phase 1 (vitest) covers framework needs._

---

## Manual-Only Verifications

| Behavior                | Requirement | Why Manual                                   | Test Instructions                                                                         |
| ----------------------- | ----------- | -------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Control protocol timing | STRM-03     | Requires live Claude CLI subprocess          | Spawn `claude -p` with stream-json, trigger a tool call, verify deny response is accepted |
| MCP tool allow behavior | TOOL-02     | Requires real MCP server registered with CLI | Configure an MCP server, verify allow response lets CLI execute the tool                  |

---

## Validation Sign-Off

- [x] All tasks have automated verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references
- [x] No watch-mode flags
- [x] Feedback latency < 3s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** approved 2026-03-15

## Validation Audit 2026-03-15

| Metric     | Count |
| ---------- | ----- |
| Gaps found | 0     |
| Resolved   | 0     |
| Escalated  | 0     |
