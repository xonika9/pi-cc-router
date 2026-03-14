---
status: complete
phase: 01-core-subprocess-bridge
source: [01-01-SUMMARY.md, 01-02-SUMMARY.md, 01-03-SUMMARY.md]
started: 2026-03-14T04:30:00Z
updated: 2026-03-14T05:05:00Z
---

## Current Test

[testing complete]

## Tests

### 1. Provider Registration
expected: Extension loads in pi and "pi-claude-cli" appears as a selectable provider. All current Claude models are listed under the provider.
result: pass

### 2. Simple Text Prompt
expected: Select a pi-claude-cli model, send a simple text prompt. Response streams back token-by-token and completes with a coherent answer.
result: issue
reported: "Response appears all at once rather than streaming token-by-token. Verified CLI does emit stream_event messages with text_delta, so events are produced -- but pi renders the complete text at once instead of incrementally."
severity: minor

### 3. Multi-Turn Conversation
expected: Send a follow-up message referencing the previous answer. Claude responds with awareness of the prior turn.
result: pass

### 4. No Orphaned Processes
expected: After a response completes, no lingering claude subprocess should remain running.
result: pass

## Summary

total: 4
passed: 3
issues: 1
pending: 0
skipped: 0

## Gaps

- truth: "Response streams back token-by-token (not all at once)"
  status: failed
  reason: "User reported: Response appears all at once rather than streaming. CLI emits stream_event messages correctly, possible pi-side rendering or readline buffering issue."
  severity: minor
  test: 2
  artifacts: []
  missing: []
