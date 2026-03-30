---
phase: 4
slug: custom-tool-mcp-proxy
status: validated
nyquist_compliant: true
wave_0_complete: true
created: 2026-03-14
---

# Phase 4 — Validation Strategy

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

| Req ID  | Requirement                                                   | Test Type | Automated Command                                                     | File Exists | Status   |
| ------- | ------------------------------------------------------------- | --------- | --------------------------------------------------------------------- | ----------- | -------- |
| MCP-01  | Custom tools discovered and filtered from built-ins           | unit      | `npx vitest run tests/mcp-config.test.ts -t "getCustomToolDefs"`      | ✅          | ✅ green |
| MCP-01  | MCP config written with correct server entry                  | unit      | `npx vitest run tests/mcp-config.test.ts -t "writeMcpConfig"`         | ✅          | ✅ green |
| MCP-01  | --mcp-config flag passed to subprocess                        | unit      | `npx vitest run tests/process-manager.test.ts -t "mcp-config"`        | ✅          | ✅ green |
| MCP-02  | mcp**custom-tools** prefix stripped from tool names           | unit      | `npx vitest run tests/tool-mapping.test.ts -t "MCP prefix stripping"` | ✅          | ✅ green |
| MCP-02  | isCustomToolName identifies non-built-in tools                | unit      | `npx vitest run tests/tool-mapping.test.ts -t "isCustomToolName"`     | ✅          | ✅ green |
| CONF-01 | No --strict-mcp-config flag (user configs load automatically) | unit      | `npx vitest run tests/process-manager.test.ts -t "strict-mcp-config"` | ✅          | ✅ green |

_Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky_

---

## Wave 0 Requirements

- [x] `tests/mcp-config.test.ts` — MCP-01 (custom tool discovery, config generation)
- [x] MCP prefix stripping tests in `tests/tool-mapping.test.ts` — MCP-02
- [x] --mcp-config and --strict-mcp-config tests in `tests/process-manager.test.ts` — MCP-01, CONF-01

_Note: Original VALIDATION.md listed `tests/mcp-server.test.ts` — the MCP server (`mcp-schema-server.cjs`) is tested indirectly through mcp-config.test.ts (config generation) and provider integration._

---

## Manual-Only Verifications

| Behavior                                             | Requirement | Why Manual                                          | Test Instructions                                                              |
| ---------------------------------------------------- | ----------- | --------------------------------------------------- | ------------------------------------------------------------------------------ |
| Custom tools appear in Claude's tool list            | MCP-01      | Requires live Claude CLI subprocess with MCP server | Run extension with a custom tool registered, verify Claude proposes calling it |
| User `.mcp.json` configs load alongside custom-tools | CONF-01     | Requires real Claude CLI with existing MCP configs  | Run extension in a project with `.mcp.json`, verify both servers are active    |

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
