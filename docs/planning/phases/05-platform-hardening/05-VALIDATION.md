---
phase: 5
slug: platform-hardening
status: validated
nyquist_compliant: true
wave_0_complete: true
created: 2026-03-14
---

# Phase 5 — Validation Strategy

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

| Req ID  | Requirement                                              | Test Type | Automated Command                                                    | File Exists | Status   |
| ------- | -------------------------------------------------------- | --------- | -------------------------------------------------------------------- | ----------- | -------- |
| PROC-02 | cross-spawn used for subprocess spawning                 | unit      | `npx vitest run tests/process-manager.test.ts -t "spawns claude"`    | ✅          | ✅ green |
| PROC-03 | Force-kill after result with 500ms grace                 | unit      | `npx vitest run tests/process-manager.test.ts -t "cleanupProcess"`   | ✅          | ✅ green |
| PROC-03 | forceKillProcess with killed/exitCode guards             | unit      | `npx vitest run tests/process-manager.test.ts -t "forceKillProcess"` | ✅          | ✅ green |
| PROC-03 | Global process registry (register/killAll)               | unit      | `npx vitest run tests/process-manager.test.ts -t "process registry"` | ✅          | ✅ green |
| PROC-04 | Subprocess crash surfaces error with stderr + exit code  | unit      | `npx vitest run tests/provider.test.ts -t "subprocess error"`        | ✅          | ✅ green |
| PROC-04 | Inactivity timeout (180s) kills and surfaces error       | unit      | `npx vitest run tests/provider.test.ts -t "inactivity timeout"`      | ✅          | ✅ green |
| PROC-04 | Abort signal sends SIGKILL (not SIGTERM)                 | unit      | `npx vitest run tests/provider.test.ts -t "abort"`                   | ✅          | ✅ green |
| HIST-02 | Image passthrough: pi-ai to Anthropic format translation | unit      | `npx vitest run tests/prompt-builder.test.ts -t "image passthrough"` | ✅          | ✅ green |
| HIST-02 | Tool result images passed through as content blocks      | unit      | `npx vitest run tests/prompt-builder.test.ts -t "tool result image"` | ✅          | ✅ green |
| HIST-02 | Non-final images get placeholder + console.warn          | unit      | `npx vitest run tests/prompt-builder.test.ts -t "placeholder"`       | ✅          | ✅ green |

_Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky_

---

## Wave 0 Requirements

- [x] `tests/process-manager.test.ts` — forceKillProcess, process registry, 500ms cleanupProcess
- [x] `tests/provider.test.ts` — subprocess crash, inactivity timeout, stderr surfacing, abort SIGKILL
- [x] `tests/prompt-builder.test.ts` — image passthrough, tool result images, placeholder text + console.warn

_Existing infrastructure covers framework setup (vitest already configured)._

---

## Manual-Only Verifications

| Behavior                                  | Requirement | Why Manual                                                    | Test Instructions                                                                       |
| ----------------------------------------- | ----------- | ------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Windows .cmd shim resolution              | PROC-02     | Requires actual Windows environment with Claude CLI installed | Run extension on Windows, verify subprocess spawns correctly                            |
| Orphan prevention on extension deactivate | PROC-03     | Requires pi runtime to trigger deactivate hook                | Start a request, close pi mid-request, check task manager for orphaned claude processes |

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
