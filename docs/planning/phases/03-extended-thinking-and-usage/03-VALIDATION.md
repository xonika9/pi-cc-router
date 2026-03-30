---
phase: 3
slug: extended-thinking-and-usage
status: validated
nyquist_compliant: true
wave_0_complete: true
created: 2026-03-14
---

# Phase 3 — Validation Strategy

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

| Req ID  | Requirement                                                | Test Type | Automated Command                                          | File Exists | Status   |
| ------- | ---------------------------------------------------------- | --------- | ---------------------------------------------------------- | ----------- | -------- |
| CONF-02 | Configurable thinking budget per model, elevated for Opus  | unit      | `npx vitest run tests/thinking-config.test.ts`             | ✅          | ✅ green |
| CONF-02 | --effort flag passed to subprocess                         | unit      | `npx vitest run tests/process-manager.test.ts -t "effort"` | ✅          | ✅ green |
| CONF-02 | Effort wired through provider to spawnClaude               | unit      | `npx vitest run tests/provider.test.ts -t "effort"`        | ✅          | ✅ green |
| STRM-04 | Thinking events bridged (thinking_start/delta/end)         | unit      | `npx vitest run tests/event-bridge.test.ts -t "thinking"`  | ✅          | ✅ green |
| STRM-05 | Usage metrics tracked from message_start and message_delta | unit      | `npx vitest run tests/event-bridge.test.ts -t "usage"`     | ✅          | ✅ green |

_Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky_

---

## Wave 0 Requirements

- [x] `tests/thinking-config.test.ts` — CONF-02 (effort tables, Opus detection, xhigh downgrade, budgets warning)
- [x] Effort flag tests in `tests/process-manager.test.ts` — CONF-02 (--effort flag appending)
- [x] Effort wiring tests in `tests/provider.test.ts` — CONF-02 (reasoning options flow-through)
- [x] Thinking event tests in `tests/event-bridge.test.ts` — STRM-04 (thinking_start/delta/end, signature)
- [x] Usage tracking tests in `tests/event-bridge.test.ts` — STRM-05 (message_start, message_delta)

---

## Manual-Only Verifications

_All phase behaviors have automated verification._

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
