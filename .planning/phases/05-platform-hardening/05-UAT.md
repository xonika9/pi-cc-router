---
status: complete
phase: 05-platform-hardening
source: [05-01-SUMMARY.md, 05-02-SUMMARY.md]
started: 2026-03-15T06:15:00Z
updated: 2026-03-15T06:50:00Z
---

## Current Test

[testing complete]

## Tests

### 1. All unit tests pass
expected: Run `npx vitest run --reporter=verbose` — all 248 tests pass with zero failures. No regressions from Phase 5 changes.
result: pass

### 2. Subprocess cleanup after response
expected: Send a prompt through pi using the Claude provider. After the response completes, check `tasklist | findstr claude` — no orphaned `claude` processes should remain. The subprocess should be killed within ~500ms of the result.
result: pass

### 3. Cancel mid-stream kills subprocess immediately
expected: Start a prompt that triggers a long response. Press Escape mid-stream to cancel. Check `tasklist | findstr claude` — no orphaned claude processes should remain.
result: pass

### 4. Image in conversation reaches Claude
expected: In pi, send a message that includes an image (screenshot, photo, etc.) as the current/latest message. Claude's response should reference or describe the image content, confirming the image data was passed through as Anthropic content blocks (not just placeholder text).
result: issue -> fixed -> pass
reported: "Image pasted in pi was saved as temp file path. Claude used read tool to get image, but on next turn toolResultContentToText() drops the image data from the tool result — only keeps text blocks. Claude never sees image pixels and hallucinates the content. Direct Claude Code correctly identifies the image."
severity: major
fix: "Updated toolResultContentToText() to handle image blocks with placeholder text. Updated buildPrompt() to check tool results for images and use ContentBlock[] path with actual image passthrough. Commit 8d9501f."
retest: pass — Claude correctly identified C# ScheduleNext() code from pasted screenshot

### 5. Historical image shows placeholder
expected: In a multi-turn conversation where an earlier message contained an image, send a follow-up text-only message. The prompt replayed to Claude should contain "[An image was shared here but could not be included]" for the historical image.
result: skipped
reason: Pi sends images as file paths (text), not as image content blocks. Placeholder logic for { type: "image" } blocks never triggers. Test invalid given Test 4 findings.

### 6. Subprocess error surfaces to user
expected: If the Claude CLI is temporarily unavailable or crashes, sending a prompt should surface a clear error message to pi's chat — including stderr content and/or exit code.
result: skipped
reason: Not practical to test manually without modifying extension code or renaming CLI binaries. Unit tests cover error surfacing paths.

### 7. Extension teardown kills all subprocesses
expected: Start a request, then close/exit pi while the response is still streaming. All `claude` subprocesses should be killed by the `process.on("exit", killAllProcesses)` hook. No orphaned processes should remain.
result: pass

## Summary

total: 7
passed: 5
issues: 0
pending: 0
skipped: 2

## Gaps

[none — issue from test 4 was fixed and retested]
