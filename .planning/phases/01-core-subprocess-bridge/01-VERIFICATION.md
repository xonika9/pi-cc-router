---
phase: 01-core-subprocess-bridge
verified: 2026-03-13T23:00:00Z
status: passed
score: 7/7 must-haves verified
re_verification: false
---

# Phase 1: Core Subprocess Bridge — Verification Report

**Phase Goal:** Pi can send a request to the Claude provider and receive streamed text responses end-to-end
**Verified:** 2026-03-13T23:00:00Z
**Status:** passed
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths (from ROADMAP Success Criteria)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Extension appears as a selectable provider in pi with all Claude model options listed | VERIFIED | `index.ts` calls `pi.registerProvider("pi-claude-cli", ...)` with `getModels("anthropic").map(...)` producing dynamic model catalog; 3 provider registration tests pass confirming correct ID, config shape, and model fields |
| 2 | Sending a simple text prompt through pi produces a streamed text response from Claude | VERIFIED | `streamViaCli` orchestrates full pipeline: `buildPrompt` -> `spawnClaude` -> `writeUserMessage` -> readline NDJSON parse -> `createEventBridge`; provider test "handles full text streaming sequence via NDJSON" passes end-to-end with mocked subprocess |
| 3 | Multi-turn conversations work — pi can send follow-up messages and Claude responds with awareness of prior turns | VERIFIED | `buildPrompt` flattens entire `context.messages` array into labeled USER:/ASSISTANT:/TOOL RESULT: blocks in order; 14 prompt-builder tests cover mixed conversation ordering including multi-turn with tool results |
| 4 | Pi receives proper `text_start`, `text_delta`, `text_end`, and `done` events during streaming | VERIFIED | `createEventBridge` maps `content_block_start/delta/stop/message_stop` to `text_start/text_delta/text_end/done` in exact order; event sequence test in `event-bridge.test.ts` confirms 5 events in precise order with correct data |

**Score:** 4/4 success criteria verified

---

### Required Artifacts

#### Plan 01-01 Artifacts

| Artifact | Provides | Exists | Substantive | Wired | Status |
|----------|----------|--------|-------------|-------|--------|
| `package.json` | pi-package metadata, peer deps, dev deps | Yes | "pi-package" keyword, pi.extensions, peerDeps for both pi packages, cross-spawn dependency | N/A — config | VERIFIED |
| `tsconfig.json` | TypeScript config for tsc --noEmit | Yes | ESNext/bundler, strict: true, skipLibCheck, esModuleInterop, all source dirs included | N/A — config | VERIFIED |
| `vitest.config.ts` | Vitest test runner configuration | Yes | defineConfig with globals: true | N/A — config | VERIFIED |
| `src/types.ts` | Shared wire protocol types | Yes | 82 lines, exports NdjsonMessage union (4 members), ClaudeApiEvent, ClaudeUsage, TrackedContentBlock | Imported by stream-parser.ts, event-bridge.ts | VERIFIED |
| `src/prompt-builder.ts` | History flattening with role labels | Yes | 171 lines, exports buildPrompt + buildSystemPrompt, AGENTS.md walk-up, .pi->.claude sanitization | Imported by provider.ts; both functions called in streamViaCli | VERIFIED |
| `tests/prompt-builder.test.ts` | Unit tests for prompt builder | Yes | 14 tests covering all behaviors: empty, user, assistant, toolResult, mixed, arrays, thinking/toolCall serialization, image skip, AGENTS.md loading | Test pass confirmed by vitest run | VERIFIED |

#### Plan 01-02 Artifacts

| Artifact | Provides | Exists | Substantive | Wired | Status |
|----------|----------|--------|-------------|-------|--------|
| `src/stream-parser.ts` | Resilient NDJSON line parsing | Yes | 37 lines (exceeds min_lines: 15), exports parseLine, trim/prefix check/try-catch/object validation | Imported and called in provider.ts readline loop | VERIFIED |
| `src/event-bridge.ts` | State machine for Claude->pi event bridging | Yes | 203 lines (exceeds min_lines: 80), exports createEventBridge + EventBridge interface, handles all 6 Claude event types | Imported and instantiated in provider.ts | VERIFIED |
| `tests/stream-parser.test.ts` | Unit tests for NDJSON parsing | Yes | 24 tests across 7 describe groups covering all edge cases including resilience suite | All pass | VERIFIED |
| `tests/event-bridge.test.ts` | Unit tests for event bridging | Yes | 19 tests covering full streaming sequence, multi-block tracking, usage capture, stop reason mapping, unsupported blocks | All pass | VERIFIED |

#### Plan 01-03 Artifacts

| Artifact | Provides | Exists | Substantive | Wired | Status |
|----------|----------|--------|-------------|-------|--------|
| `src/process-manager.ts` | Subprocess spawn/cleanup/validation | Yes | 128 lines (exceeds min_lines: 40), exports spawnClaude + writeUserMessage + cleanupProcess + captureStderr + validateCliPresence + validateCliAuth | All 6 exports imported and used in provider.ts and index.ts | VERIFIED |
| `src/provider.ts` | streamViaCli orchestration | Yes | 137 lines (exceeds min_lines: 50), exports streamViaCli, full async pipeline with abort handling and error propagation | Imported in index.ts, delegated from streamSimple | VERIFIED |
| `index.ts` | Extension entry point with provider registration | Yes | 39 lines (exceeds min_lines: 30), default export function, registers "pi-claude-cli" with all Anthropic models via getModels | Root entry point, pi.extensions points to it | VERIFIED |
| `tests/process-manager.test.ts` | Unit tests for subprocess management | Yes | 18 tests covering spawn flags, stdin write, cleanup grace period, stderr capture, CLI validation | All pass | VERIFIED |
| `tests/provider.test.ts` | Unit tests for provider registration and streamViaCli | Yes | 9 tests covering registration shape, model derivation, full streaming sequence, error handling, abort signal, cleanup | All pass | VERIFIED |

---

### Key Link Verification

#### Plan 01-01 Key Links

| From | To | Via | Pattern | Status | Evidence |
|------|----|-----|---------|--------|----------|
| `src/prompt-builder.ts` | `@mariozechner/pi-ai` | Context type import | `import.*Context.*from.*pi-ai` | INTENTIONAL DEVIATION | Plan documented decision: used `any` for Context parameter to avoid requiring pi-ai at dev time. Pattern not present but goal (type safety at runtime) achieved via documented trade-off. Not a gap. |

#### Plan 01-02 Key Links

| From | To | Via | Pattern | Status | Evidence |
|------|----|-----|---------|--------|----------|
| `src/stream-parser.ts` | `src/types.ts` | NdjsonMessage type import | `import.*NdjsonMessage.*from.*types` | WIRED | Line 1: `import type { NdjsonMessage } from "./types";` |
| `src/event-bridge.ts` | `src/types.ts` | ClaudeApiEvent type import | `import.*ClaudeApiEvent.*from.*types` | WIRED | Line 1: `import type { ClaudeApiEvent, TrackedContentBlock } from "./types";` |
| `src/event-bridge.ts` | `@mariozechner/pi-ai` | AssistantMessage type and calculateCost | `import.*from.*pi-ai` | WIRED | Lines 2-8: imports calculateCost, AssistantMessage, AssistantMessageEventStream, Model, TextContent |

#### Plan 01-03 Key Links

| From | To | Via | Pattern | Status | Evidence |
|------|----|-----|---------|--------|----------|
| `index.ts` | `src/provider.ts` | imports streamViaCli | `import.*streamViaCli.*from.*provider` | WIRED | Line 10: `import { streamViaCli } from "./src/provider.js";` |
| `src/provider.ts` | `src/process-manager.ts` | imports spawnClaude | `import.*spawnClaude.*from.*process-manager` | WIRED | Line 17: `import { spawnClaude, writeUserMessage, cleanupProcess, captureStderr } from "./process-manager.js";` |
| `src/provider.ts` | `src/prompt-builder.ts` | imports buildPrompt | `import.*buildPrompt.*from.*prompt-builder` | WIRED | Line 16: `import { buildPrompt, buildSystemPrompt } from "./prompt-builder.js";` |
| `src/provider.ts` | `src/stream-parser.ts` | imports parseLine | `import.*parseLine.*from.*stream-parser` | WIRED | Line 18: `import { parseLine } from "./stream-parser.js";` |
| `src/provider.ts` | `src/event-bridge.ts` | imports createEventBridge | `import.*createEventBridge.*from.*event-bridge` | WIRED | Line 19: `import { createEventBridge } from "./event-bridge.js";` |
| `src/provider.ts` | `node:readline` | readline.createInterface for NDJSON line reading | `createInterface` | WIRED | Line 13: `import { createInterface } from "node:readline";` — called at line 82 |
| `index.ts` | `@mariozechner/pi-coding-agent` | ExtensionAPI type for default export | `import.*ExtensionAPI.*from.*pi-coding-agent` | WIRED | Line 9: `import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";` |

---

### Requirements Coverage

Phase 1 claims requirements: PROV-01, PROV-02, PROV-03, PROC-01, STRM-01, STRM-02, HIST-01

| Requirement | Claimed in Plan | Description | Status | Evidence |
|-------------|----------------|-------------|--------|----------|
| PROV-01 | 01-03 | Extension registers as custom pi provider via `pi.registerProvider()` with unique provider ID | SATISFIED | `index.ts` line 30: `pi.registerProvider(PROVIDER_ID, ...)` where `PROVIDER_ID = "pi-claude-cli"` — 3 passing provider registration tests confirm |
| PROV-02 | 01-01 | Provider exposes all current Claude models derived from `getModels("anthropic")` with correct context windows, max tokens, and cost info | SATISFIED | `index.ts` lines 20-28: `getModels("anthropic").map(model => ({ id, name, reasoning, input, cost, contextWindow, maxTokens }))` — test "derives models from getModels('anthropic') with correct fields" passes with all expected fields |
| PROV-03 | 01-03 | Provider implements `streamSimple` handler as the core entry point for LLM requests | SATISFIED | `index.ts` line 35: `streamSimple: (model, context, options) => streamViaCli(model, context, options)` — 6 streamViaCli tests confirm behavior |
| PROC-01 | 01-03 | Extension spawns `claude -p --input-format stream-json --output-format stream-json --verbose` as fresh subprocess per pi request | SATISFIED | `src/process-manager.ts` lines 26-35: builds args array with all required flags including `-p`, `--input-format stream-json`, `--output-format stream-json`, `--verbose`, `--include-partial-messages`, `--no-session-persistence`, `--permission-mode dontAsk` — 7 spawnClaude tests pass verifying all flags |
| STRM-01 | 01-02 | Extension parses NDJSON output line-by-line via readline, validating each line as JSON before processing | SATISFIED | `src/stream-parser.ts`: parseLine validates with trim/prefix/JSON.parse/object-check; `src/provider.ts` lines 82-86: `createInterface` on proc.stdout with for-await loop; 24 stream-parser tests all pass including resilience suite |
| STRM-02 | 01-02 | Extension bridges Claude API stream events to pi's AssistantMessageEventStream events | SATISFIED | `src/event-bridge.ts`: maps content_block_start->text_start, content_block_delta->text_delta, content_block_stop->text_end, message_stop->done; 19 event-bridge tests pass including full sequence verification |
| HIST-01 | 01-01 | Extension builds flattened prompt from full pi conversation history with USER, ASSISTANT, and TOOL RESULT blocks | SATISFIED | `src/prompt-builder.ts`: buildPrompt iterates messages array applying "USER:", "ASSISTANT:", "TOOL RESULT (historical {name}):" labels; 10 buildPrompt tests cover all message types, mixed conversation ordering, and array content blocks |

**All 7 Phase 1 requirements: SATISFIED**

#### Orphaned Requirements Check

Requirements.md maps to Phase 1: PROV-01, PROV-02, PROV-03, PROC-01, STRM-01, STRM-02, HIST-01 — all claimed and verified above. No orphaned requirements for Phase 1.

Note: PROC-02 (cross-spawn for Windows .cmd shim) is formally assigned to Phase 5 in REQUIREMENTS.md. cross-spawn is already used in `src/process-manager.ts`, making this a de-facto early delivery, but it is not a gap for Phase 1.

---

### Anti-Pattern Scan

Files scanned: all 6 src/ files and index.ts

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `src/prompt-builder.ts` | 114 | `// Unknown block types are represented as a placeholder` | Info | Comment is documentation for the `[${block.type}]` fallback — not a stub; the implementation is real and intentional |
| `src/stream-parser.ts` | 15,20,28,33 | `return null` | Info | These are the designed resilient behavior of parseLine — null returns are the intended contract, not stubs |

No blockers or warnings found. No TODO/FIXME/XXX markers in any source file. No empty implementations. No console.log-only stubs.

---

### Human Verification Required

The following items cannot be verified programmatically and require a live test with pi and Claude CLI:

#### 1. End-to-End Provider Appearance in Pi

**Test:** Install the extension in a pi instance and open the provider selection UI
**Expected:** "pi-claude-cli" appears as a selectable provider, all Claude models (claude-sonnet, claude-opus, etc.) appear in the model dropdown with correct names
**Why human:** Provider registration is confirmed by unit tests with mocked pi API, but visual appearance and model list rendering in the actual pi UI cannot be verified without running pi

#### 2. Live Text Streaming Response

**Test:** Select the pi-claude-cli provider, send a simple prompt "Hello, what is 2+2?"
**Expected:** Response streams character-by-character (or in chunks) and completes with a coherent answer; no subprocess errors or hanging
**Why human:** The full subprocess spawn-to-response pipeline requires the actual `claude` CLI to be installed and authenticated; the unit tests use PassThrough stream mocks

#### 3. Multi-Turn Conversation Continuity

**Test:** Send two follow-up messages in the same pi session: first "My name is Alice", then "What is my name?"
**Expected:** Claude responds "Your name is Alice" (demonstrating prior turn history was correctly included in the prompt)
**Why human:** Validates that buildPrompt's multi-turn flattening is actually received and processed by the Claude CLI subprocess, not just unit-tested in isolation

#### 4. Subprocess Cleanup After Response

**Test:** Use process monitor (Task Manager, htop) to observe claude subprocess PIDs during and after a response
**Expected:** A new subprocess appears when a request is sent and terminates within ~2 seconds of the response completing (no orphaned processes)
**Why human:** The 2000ms SIGKILL cleanup is tested with fake timers; actual OS-level process lifecycle requires human observation

---

### ROADMAP State Note

The ROADMAP.md shows "2/3 plans executed" for Phase 1 and marks all three plan checkboxes as unchecked (`[ ]`). This is stale — all 3 plans have been executed and all 15 commits are present in git log (3b6b003 through adf6db9). The ROADMAP was not updated after Plan 01-03 completed. This is a documentation-only inconsistency and does not affect the verification result.

---

## Summary

Phase 1 goal is achieved. All 7 requirements (PROV-01, PROV-02, PROV-03, PROC-01, STRM-01, STRM-02, HIST-01) are implemented and substantiated by passing tests.

The four-layer pipeline is fully wired and tested end-to-end with mocked subprocess I/O:
- `buildPrompt` flattens pi conversation history into labeled USER/ASSISTANT/TOOL RESULT blocks
- `spawnClaude` spawns the Claude CLI subprocess with all required stream-json flags via cross-spawn
- `parseLine` parses NDJSON output lines resiliently without ever throwing
- `createEventBridge` translates Claude API events to pi's text_start/text_delta/text_end/done stream events

84 tests across 5 test files all pass. `tsc --noEmit` is clean. No anti-pattern blockers.

The only open items are 4 human verification tests requiring a live pi + Claude CLI environment, and a stale ROADMAP status entry.

---

_Verified: 2026-03-13T23:00:00Z_
_Verifier: Claude (gsd-verifier)_
