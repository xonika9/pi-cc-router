---
phase: 1
slug: core-subprocess-bridge
status: validated
nyquist_compliant: true
wave_0_complete: true
created: 2026-03-13
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property               | Value                               |
| ---------------------- | ----------------------------------- |
| **Framework**          | vitest ^3.0.0                       |
| **Config file**        | vitest.config.ts                    |
| **Quick run command**  | `npx vitest run --reporter=verbose` |
| **Full suite command** | `npx vitest run`                    |
| **Estimated runtime**  | ~3 seconds                          |

---

## Sampling Rate

- **After every task commit:** Run `npx vitest run --reporter=verbose`
- **After every plan wave:** Run `npx vitest run`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 3 seconds

---

## Per-Task Verification Map

| Req ID  | Requirement                                                     | Test Type         | Automated Command                                                                   | File Exists | Status   |
| ------- | --------------------------------------------------------------- | ----------------- | ----------------------------------------------------------------------------------- | ----------- | -------- |
| PROV-01 | Provider registers with correct ID and streamSimple handler     | unit              | `npx vitest run tests/provider.test.ts -t "registers provider"`                     | ✅          | ✅ green |
| PROV-02 | Models derived from getModels("anthropic") with correct fields  | unit              | `npx vitest run tests/provider.test.ts -t "derives models"`                         | ✅          | ✅ green |
| PROV-03 | streamSimple returns AssistantMessageEventStream                | unit              | `npx vitest run tests/provider.test.ts -t "returns an AssistantMessageEventStream"` | ✅          | ✅ green |
| PROC-01 | Subprocess spawned with correct flags                           | unit (mock spawn) | `npx vitest run tests/process-manager.test.ts -t "spawns claude"`                   | ✅          | ✅ green |
| STRM-01 | NDJSON parsing handles valid JSON, malformed lines, debug noise | unit              | `npx vitest run tests/stream-parser.test.ts`                                        | ✅          | ✅ green |
| STRM-02 | Text stream events bridged correctly (text_start/delta/end)     | unit              | `npx vitest run tests/event-bridge.test.ts -t "text content block"`                 | ✅          | ✅ green |
| HIST-01 | Flattened prompt built from conversation history                | unit              | `npx vitest run tests/prompt-builder.test.ts -t "buildPrompt"`                      | ✅          | ✅ green |

_Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky_

---

## Wave 0 Requirements

- [x] `vitest` + `@types/node` — dev dependencies
- [x] `vitest.config.ts` — Vitest configuration file
- [x] `tests/provider.test.ts` — PROV-01, PROV-02, PROV-03
- [x] `tests/process-manager.test.ts` — PROC-01
- [x] `tests/stream-parser.test.ts` — STRM-01
- [x] `tests/event-bridge.test.ts` — STRM-02
- [x] `tests/prompt-builder.test.ts` — HIST-01

---

## Manual-Only Verifications

| Behavior                              | Requirement | Why Manual                    | Test Instructions                                |
| ------------------------------------- | ----------- | ----------------------------- | ------------------------------------------------ |
| Extension appears in pi provider list | PROV-01     | Requires running pi runtime   | Load extension in pi, verify provider appears    |
| End-to-end text streaming             | STRM-02     | Requires live Claude CLI + pi | Send prompt through pi, verify streamed response |
| Multi-turn conversation               | HIST-01     | Requires live Claude CLI + pi | Send multiple turns, verify context awareness    |

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
