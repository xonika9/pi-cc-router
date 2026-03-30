---
phase: 6
slug: testing-and-release-pipeline
status: validated
nyquist_compliant: true
wave_0_complete: true
created: 2026-03-15
---

# Phase 6 — Validation Strategy

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

| Req ID  | Requirement                                       | Test Type | Automated Command                                                                                                                                 | File Exists    | Status   |
| ------- | ------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------- | -------------- | -------- |
| RLSE-01 | Unit tests cover all modules with mocked I/O      | unit      | `npx vitest run --reporter=verbose`                                                                                                               | ✅ (270 tests) | ✅ green |
| RLSE-01 | ESLint passes                                     | smoke     | `npx eslint .`                                                                                                                                    | ✅             | ✅ green |
| RLSE-02 | CI workflow runs lint, typecheck, tests on 3 OSes | config    | `cat .github/workflows/ci.yml`                                                                                                                    | ✅             | ✅ green |
| RLSE-03 | Publish workflow triggers on v\* tags             | config    | `cat .github/workflows/publish.yml`                                                                                                               | ✅             | ✅ green |
| RLSE-04 | package.json has pi-package keyword               | unit      | `node -e "require('assert')(require('./package.json').keywords.includes('pi-package'))"`                                                          | ✅             | ✅ green |
| RLSE-04 | package.json has pi.extensions entry              | unit      | `node -e "require('assert')(require('./package.json').pi.extensions.length > 0)"`                                                                 | ✅             | ✅ green |
| RLSE-04 | package.json has correct peer dependencies        | unit      | `node -e "const p=require('./package.json').peerDependencies; require('assert')(p['@mariozechner/pi-ai'] && p['@mariozechner/pi-coding-agent'])"` | ✅             | ✅ green |

_Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky_

---

## Wave 0 Requirements

- [x] `eslint.config.mjs` — ESLint flat config with typescript-eslint
- [x] `.github/workflows/ci.yml` — CI workflow with matrix [ubuntu, windows, macos]
- [x] `.github/workflows/publish.yml` — Publish workflow on v\* tags with npm publish --provenance
- [x] 270 unit tests across 9 test files — all green

---

## Manual-Only Verifications

| Behavior                        | Requirement | Why Manual                                   | Test Instructions                                                          |
| ------------------------------- | ----------- | -------------------------------------------- | -------------------------------------------------------------------------- |
| Tagged release publishes to npm | RLSE-03     | Requires npm credentials and actual tag push | Push a `v*` tag, verify npm publish succeeds and GitHub Release is created |
| CI runs on all 3 OS             | RLSE-02     | Requires actual GitHub Actions execution     | Open PR, verify jobs pass on windows-latest, ubuntu-latest, macos-latest   |

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
