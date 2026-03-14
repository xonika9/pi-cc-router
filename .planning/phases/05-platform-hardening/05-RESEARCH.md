# Phase 5: Platform Hardening - Research

**Researched:** 2026-03-14
**Domain:** Cross-platform subprocess lifecycle, error surfacing, image history replay
**Confidence:** HIGH

## Summary

Phase 5 covers four requirements: PROC-02 (cross-spawn for Windows, already satisfied), PROC-03 (force-kill after result), PROC-04 (error surfacing from subprocess failures), and HIST-02 (image replay in conversation history). The codebase already has solid foundations -- `cross-spawn` is imported and used, stderr capture exists, error events follow an established shape. The work is primarily about hardening existing code paths and adding new ones for edge cases.

The key technical findings are: (1) `proc.kill('SIGKILL')` works on Windows -- Node.js treats all kill signals as forceful termination on Windows, so no `taskkill` shim is needed; (2) the Claude CLI `stream-json` input format likely accepts the Anthropic API image content block format (`{ type: "image", source: { type: "base64", media_type, data } }`), but pi-ai uses a different shape (`{ type: "image", data, mimeType }`) requiring translation; (3) extended thinking `thinking_delta` events stream in chunks with possible multi-minute gaps, making inactivity timeout design critical -- a 120-second minimum is recommended to avoid false triggers.

**Primary recommendation:** Implement platform-aware kill, inactivity timeout, subprocess exit handling, global process registry, and image placeholder logic as incremental additions to existing `process-manager.ts`, `provider.ts`, `prompt-builder.ts`, and `index.ts`.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Include stderr output and exit code in error events -- developers can debug, non-technical users can report
- Buffer stderr, only surface it when the process exits abnormally (not real-time to console)
- If partial text was already streamed before a crash, keep the partial response and append an error event at the end -- don't discard useful output
- Placeholder text inserted when images are dropped: "[An image was shared here but could not be included]"
- Console.warn once per request when images are dropped from history (not per image) -- follows existing warn-don't-block pattern
- Use an inactivity timeout (no stdout output for N seconds), not a total request timeout -- long responses are fine as long as tokens keep flowing
- Hardcoded default duration, not configurable -- keep it simple, can add configurability later if users ask
- When timeout fires: kill subprocess and emit error event with clear message
- Platform-aware kill strategy: taskkill /F on Windows, SIGKILL on Unix -- ensures no orphans on any platform
- On abort signal (user cancels): immediate force-kill, no grace period -- user wants it stopped now
- Global process registry: keep a Set of active child processes, kill all on extension deactivate or process exit -- prevents orphaned claude processes
- Grace period after result event: reduce from 2000ms to 500ms -- we have everything we need from the result, brief buffer for edge-case stdout flushing

### Claude's Discretion
- Exact wording of error messages for unknown failures (no stderr, non-zero exit code)
- Exact inactivity timeout duration (recommended: 120-300 seconds based on analysis)
- Implementation details of the process registry (Set vs WeakRef, cleanup timing)
- Platform detection approach for kill strategy

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| PROC-02 | Extension uses cross-spawn for subprocess creation to handle Windows .cmd shim resolution | Already satisfied in `process-manager.ts` line 9, 50. cross-spawn is imported and used for all subprocess spawning. Verify only. |
| PROC-03 | Extension force-kills the subprocess after receiving the result event (known CLI hang bug) | `cleanupProcess()` exists at line 82 but needs grace period reduction (2000ms to 500ms) and platform-aware kill. Research covers Node.js kill behavior on Windows. |
| PROC-04 | Extension propagates subprocess crashes, timeouts, and stderr errors to pi as error events | Error event shape established (`{ type: "error", reason: "error", error: message }`). Research covers inactivity timeout, subprocess exit/close handlers, and thinking_delta gap behavior. |
| HIST-02 | Extension includes base64-encoded images in the replayed prompt when present in pi conversation messages | Research covers pi-ai image format (`{ type: "image", data, mimeType }`), Anthropic API format, and stream-json input compatibility. Fallback to placeholder decided by user. |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| cross-spawn | ^7.0.6 | Cross-platform subprocess spawning | Already in use. Handles Windows .cmd shim resolution transparently. |
| node:child_process | Built-in | ChildProcess type, kill(), event handlers | Standard Node.js API for subprocess lifecycle management. |
| node:process | Built-in | `process.platform`, `process.on('exit')` | Platform detection and cleanup hooks. |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| node:child_process (execSync) | Built-in | `taskkill /F /PID` on Windows | Only needed if proc.kill() proves insufficient for Windows process trees (see research note below). |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Manual taskkill | tree-kill npm package | Adds dependency; proc.kill('SIGKILL') already works on Windows for direct children |
| Configurable timeout | Hardcoded timeout | User decided: hardcoded for now, configurable later if needed |

**Installation:**
No new dependencies needed. All required packages are already installed.

## Architecture Patterns

### Recommended Project Structure
```
src/
  process-manager.ts   # + forceKillProcess(), processRegistry, platform-aware kill
  provider.ts          # + inactivity timeout, subprocess exit/close handlers, registry integration
  prompt-builder.ts    # + image placeholder in userContentToText()
index.ts               # + process registry cleanup on deactivate
```

### Pattern 1: Platform-Aware Force Kill
**What:** A function that kills a subprocess using the appropriate mechanism for the current platform.
**When to use:** Every subprocess cleanup path (result received, timeout, abort, deactivate).
**Example:**
```typescript
// Source: Node.js v25.8.1 docs - subprocess.kill() on Windows
// On Windows, proc.kill('SIGKILL') works -- Node.js treats ALL kill signals
// as forceful abrupt termination on Windows since POSIX signals don't exist.
// This means proc.kill('SIGKILL') is cross-platform safe.
export function forceKillProcess(proc: ChildProcess): void {
  if (proc.killed || proc.exitCode !== null) return;
  proc.kill("SIGKILL");
}
```

**IMPORTANT RESEARCH NOTE on proc.kill('SIGKILL') vs taskkill:**
The CONTEXT.md specifies "taskkill /F on Windows, SIGKILL on Unix." However, Node.js documentation states that on Windows, `proc.kill('SIGKILL')` already "kills the process forcefully and abruptly (similar to SIGKILL)" because Windows doesn't have POSIX signals. The only reason to use `taskkill /F /T /PID` explicitly would be to kill the **process tree** (child processes of the subprocess). Since we spawn `claude` directly (not through a shell), there is unlikely to be a process tree to worry about. **Recommendation:** Use `proc.kill('SIGKILL')` everywhere -- it is cross-platform. Add a platform-specific `taskkill /F /T /PID` fallback ONLY if testing reveals orphaned grandchild processes on Windows. Confidence: HIGH.

### Pattern 2: Inactivity Timeout with Reset on Any Stdout
**What:** Timer that resets each time any stdout data arrives; fires only when no output for N seconds.
**When to use:** Applied per-request in `streamViaCli()`.
**Example:**
```typescript
// Reset inactivity timer on each NDJSON line
let inactivityTimer: ReturnType<typeof setTimeout>;
const INACTIVITY_TIMEOUT_MS = 180_000; // 3 minutes

function resetInactivityTimer() {
  clearTimeout(inactivityTimer);
  inactivityTimer = setTimeout(() => {
    // Timeout fired -- no output for INACTIVITY_TIMEOUT_MS
    forceKillProcess(proc);
    stream.push({
      type: "error",
      reason: "error",
      error: `Claude CLI subprocess timed out: no output for ${INACTIVITY_TIMEOUT_MS / 1000} seconds`,
    } as any);
    stream.end();
  }, INACTIVITY_TIMEOUT_MS);
}

// In the readline 'line' handler:
rl.on("line", (line: string) => {
  resetInactivityTimer();
  // ... existing line processing
});

// Start the timer after writing the user message
resetInactivityTimer();
```

### Pattern 3: Global Process Registry
**What:** A `Set<ChildProcess>` that tracks all active subprocesses and kills them all on extension teardown.
**When to use:** Extension deactivation, process exit.
**Example:**
```typescript
// In process-manager.ts
const activeProcesses = new Set<ChildProcess>();

export function registerProcess(proc: ChildProcess): void {
  activeProcesses.add(proc);
  proc.on("exit", () => activeProcesses.delete(proc));
}

export function killAllProcesses(): void {
  for (const proc of activeProcesses) {
    forceKillProcess(proc);
  }
  activeProcesses.clear();
}

// In index.ts -- register cleanup hooks
process.on("exit", killAllProcesses);
// Also call killAllProcesses() from extension deactivate if pi supports it
```

### Pattern 4: Subprocess Exit Handler with Stderr Surfacing
**What:** Handle the subprocess `close` event to detect crashes and surface buffered stderr.
**When to use:** Every subprocess -- catches cases where the process exits without sending a `result` message.
**Example:**
```typescript
// In provider.ts, after spawning
proc.on("close", (code: number | null, signal: string | null) => {
  clearTimeout(inactivityTimer);
  if (code !== 0 && code !== null) {
    const stderr = getStderr();
    const message = stderr
      ? `Claude CLI exited with code ${code}: ${stderr.trim()}`
      : `Claude CLI exited unexpectedly with code ${code}`;
    stream.push({ type: "error", reason: "error", error: message } as any);
    stream.end();
  }
});
```

### Anti-Patterns to Avoid
- **Total request timeout:** Don't use a fixed wall-clock timeout. Long responses are valid as long as tokens keep flowing. The user explicitly decided against this.
- **Real-time stderr streaming:** Don't pipe stderr to console during normal operation. Buffer it and surface only on abnormal exit. Normal Claude CLI writes diagnostic info to stderr that would be confusing to users.
- **SIGTERM for abort:** Don't use SIGTERM for user cancellation. The current code uses SIGTERM (provider.ts line 72) which doesn't work reliably on Windows. Switch to immediate force-kill.
- **WeakRef for process registry:** Don't use WeakRef -- subprocess references must stay strong so we can kill them. Use a plain Set with cleanup on 'exit' event.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Windows .cmd resolution | Custom .cmd detection | cross-spawn (already used) | Handles .exe vs .cmd detection, argument quoting, path resolution |
| Process killing cross-platform | Platform-branching exec of taskkill | proc.kill('SIGKILL') | Node.js already handles this -- on Windows all signals cause forceful termination |
| Inactivity timeout | Custom interval polling | setTimeout + clearTimeout | Standard pattern, well-understood, timer reset on each line event |

**Key insight:** Node.js abstracts most platform differences for subprocess management. The main area where custom handling is needed is the error surfacing logic (translating exit codes and stderr into user-friendly error events).

## Common Pitfalls

### Pitfall 1: Inactivity Timeout False Triggers During Extended Thinking
**What goes wrong:** Claude enters an extended thinking phase with thinking_delta events streaming in chunks. Between chunks, there can be multi-minute gaps (documented up to 3+ minutes). An inactivity timeout set too low will kill the subprocess during normal thinking.
**Why it happens:** Extended thinking streams in "chunky" batches for performance. The Anthropic documentation explicitly states "delays between streaming events" are expected. Additionally, there are documented cases of text_delta pauses of 185+ seconds even outside thinking phases (see SDK issue #44).
**How to avoid:** Set inactivity timeout to at least 180 seconds (3 minutes). The timer resets on ANY stdout line, including thinking_delta events, so thinking that produces continuous deltas will never trigger it. The danger is during gaps between thinking chunks.
**Warning signs:** Users report "timeout" errors during complex prompts that work fine with simple ones.

### Pitfall 2: Stream Not Ended After Error
**What goes wrong:** An error event is pushed to the stream but `stream.end()` is not called. The pi UI hangs waiting for more events.
**Why it happens:** Multiple error paths (process error, close, timeout, abort) each need to call `stream.end()`. Missing it in one path creates a subtle bug.
**How to avoid:** Use a guard flag (`let streamEnded = false`) and a helper function that checks it before pushing error and calling end(). Every error path calls the same helper.
**Warning signs:** pi UI shows spinner indefinitely after an error message appears.

### Pitfall 3: Double Error Events
**What goes wrong:** Both the `close` handler and the `error` handler fire for the same failure, pushing two error events and calling `stream.end()` twice.
**Why it happens:** Node.js ChildProcess can emit both 'error' and 'close' events for the same failure (e.g., spawn failure). The 'close' handler fires because the streams closed, and 'error' fires because the spawn failed.
**How to avoid:** Use the same `streamEnded` guard flag. First error path wins, subsequent ones are no-ops.
**Warning signs:** Duplicate error messages in pi's chat, or "stream already ended" errors in console.

### Pitfall 4: Orphaned Processes on Extension Deactivate
**What goes wrong:** User closes the editor or disables the extension while a Claude subprocess is running. The subprocess continues running in the background consuming resources.
**Why it happens:** No cleanup hook registered for extension teardown.
**How to avoid:** Global process registry with `process.on('exit', killAllProcesses)`. Register each subprocess, remove on natural exit.
**Warning signs:** Multiple `claude` processes visible in task manager after closing the editor.

### Pitfall 5: Timer Leaks on Normal Completion
**What goes wrong:** The inactivity timeout timer is not cleared when the request completes normally. The timer fires later, attempts to kill an already-dead process, and may emit a spurious error.
**Why it happens:** `clearTimeout` not called in the result handler path.
**How to avoid:** Clear the inactivity timer in every completion path: result received, subprocess close, abort signal. The `finally` block or the guard flag approach handles this.
**Warning signs:** Spurious "timed out" errors appearing after successful responses.

## Code Examples

Verified patterns from official sources and existing codebase:

### Image Placeholder in userContentToText()
```typescript
// Source: CONTEXT.md locked decision + pi-ai image block shape
// pi-ai image blocks: { type: "image", data: string (base64), mimeType: string }
function userContentToText(content: string | any[]): string {
  if (typeof content === "string") return content;
  if (!Array.isArray(content)) return "";

  const texts: string[] = [];
  let imageCount = 0;
  for (const block of content) {
    if (block.type === "text") {
      texts.push(block.text ?? "");
    } else if (block.type === "image") {
      texts.push("[An image was shared here but could not be included]");
      imageCount++;
    }
    // Unknown block types silently skipped
  }

  if (imageCount > 0) {
    console.warn(
      `[pi-claude-cli] ${imageCount} image(s) in conversation history could not be included in the prompt`
    );
  }

  return texts.join("\n");
}
```

### Updated cleanupProcess with 500ms Grace Period
```typescript
// Source: Existing process-manager.ts, updated per CONTEXT.md decision
export function cleanupProcess(proc: ChildProcess): void {
  setTimeout(() => {
    if (!proc.killed) {
      proc.kill("SIGKILL");
    }
  }, 500); // Reduced from 2000ms -- we have everything we need from the result
}
```

### Abort Handler with Immediate Force-Kill
```typescript
// Source: Existing provider.ts abort handler, updated per CONTEXT.md decision
if (options?.signal) {
  abortHandler = () => {
    if (proc && !proc.killed) {
      proc.kill("SIGKILL"); // Changed from SIGTERM -- immediate force-kill, no grace period
    }
  };
  // ...
}
```

### Anthropic API Image Content Block (for future reference)
```typescript
// Source: Anthropic Vision API docs (platform.claude.com/docs/en/build-with-claude/vision)
// If stream-json input accepts full Messages API content blocks, this is the format:
{
  type: "image",
  source: {
    type: "base64",
    media_type: "image/jpeg", // or image/png, image/gif, image/webp
    data: "<base64-encoded-string>"
  }
}
// pi-ai format (different shape):
{
  type: "image",
  data: "<base64-encoded-string>",
  mimeType: "image/jpeg"
}
// Translation needed if we ever send images directly to CLI
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| proc.kill('SIGTERM') for abort | proc.kill('SIGKILL') for abort | This phase | SIGTERM unreliable on Windows; SIGKILL is forceful on all platforms |
| 2000ms cleanup grace period | 500ms cleanup grace period | This phase | Faster cleanup, less orphan risk |
| Images silently skipped | Image placeholder text | This phase | Claude knows visual context existed even without seeing the image |
| No timeout | Inactivity timeout | This phase | Detects stuck/crashed subprocesses |
| No process registry | Global Set<ChildProcess> | This phase | Prevents orphans on extension teardown |

**Deprecated/outdated:**
- SIGTERM for process cleanup on Windows: Always forcefully terminates anyway (Node.js behavior), but semantically wrong -- use SIGKILL for clarity.

## Open Questions

1. **Can stream-json input accept multipart content blocks with images?**
   - What we know: The Claude CLI `--input-format stream-json` accepts NDJSON messages with `{ type: "user", message: { role: "user", content: "..." } }`. The content field is currently always a string in our implementation.
   - What's unclear: Whether the content field can be an array of content blocks (like the Messages API) including image blocks. The stream-json input format documentation does not specify this. The current code sends a flattened text string, not structured content blocks.
   - Recommendation: Use placeholder approach (decided by user). If future versions need actual image passthrough, test whether stream-json accepts `content: [{ type: "text", text: "..." }, { type: "image", source: { ... } }]` format. This is a v2 investigation. Confidence: LOW for image passthrough, HIGH for placeholder approach.

2. **Exact inactivity timeout duration**
   - What we know: Extended thinking gaps can be 3+ minutes. Text delta pauses of 185 seconds have been documented. pi-claude-cli denies built-in tools (no subagent spawning), so the primary gap source is thinking phases.
   - What's unclear: Maximum possible gap between stdout events in practice.
   - Recommendation: 180 seconds (3 minutes). This covers documented worst cases with a safety margin. If users report false triggers, increase to 300 seconds. Confidence: MEDIUM.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | vitest ^3.0.0 |
| Config file | vitest.config.ts |
| Quick run command | `npx vitest run --reporter=verbose` |
| Full suite command | `npx vitest run --reporter=verbose` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| PROC-02 | cross-spawn used for subprocess spawning | unit | `npx vitest run tests/process-manager.test.ts -t "spawns claude" -x` | Yes (already tested) |
| PROC-03 | Force-kill after result with 500ms grace | unit | `npx vitest run tests/process-manager.test.ts -t "cleanupProcess" -x` | Yes (needs update for 500ms) |
| PROC-03 | Platform-aware kill function | unit | `npx vitest run tests/process-manager.test.ts -t "forceKill" -x` | No -- Wave 0 |
| PROC-04 | Subprocess crash surfaces error event | unit | `npx vitest run tests/provider.test.ts -t "crash" -x` | No -- Wave 0 |
| PROC-04 | Inactivity timeout surfaces error event | unit | `npx vitest run tests/provider.test.ts -t "timeout" -x` | No -- Wave 0 |
| PROC-04 | Stderr included in error events on abnormal exit | unit | `npx vitest run tests/provider.test.ts -t "stderr" -x` | No -- Wave 0 |
| HIST-02 | Image blocks replaced with placeholder text | unit | `npx vitest run tests/prompt-builder.test.ts -t "image" -x` | Yes (needs update for placeholder) |
| HIST-02 | Console.warn emitted once per request with image count | unit | `npx vitest run tests/prompt-builder.test.ts -t "warn" -x` | No -- Wave 0 |

### Sampling Rate
- **Per task commit:** `npx vitest run --reporter=verbose`
- **Per wave merge:** `npx vitest run --reporter=verbose`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `tests/process-manager.test.ts` -- update existing cleanupProcess test for 500ms grace period
- [ ] `tests/process-manager.test.ts` -- add forceKillProcess and process registry tests
- [ ] `tests/provider.test.ts` -- add subprocess crash, timeout, and stderr error event tests
- [ ] `tests/prompt-builder.test.ts` -- update image skip test to verify placeholder text and console.warn

## Sources

### Primary (HIGH confidence)
- [Node.js v25.8.1 child_process docs](https://nodejs.org/api/child_process.html) - subprocess.kill() Windows behavior, signal handling
- [Anthropic Vision API docs](https://platform.claude.com/docs/en/build-with-claude/vision) - image content block JSON structure, supported formats
- [Anthropic Extended Thinking docs](https://platform.claude.com/docs/en/build-with-claude/extended-thinking) - thinking_delta streaming behavior, chunky delivery pattern
- Existing codebase: `process-manager.ts`, `provider.ts`, `prompt-builder.ts`, `event-bridge.ts`, `index.ts`

### Secondary (MEDIUM confidence)
- [Claude Agent SDK Issue #44](https://github.com/anthropics/claude-agent-sdk-typescript/issues/44) - documented 185-second text_delta pause, no ping events during gap
- [Claude Code CLI reference](https://code.claude.com/docs/en/cli-reference) - stream-json flag documentation
- [cross-spawn npm](https://www.npmjs.com/package/cross-spawn) - Windows .cmd shim resolution mechanism
- [pi-mono GitHub](https://github.com/badlogic/pi-mono/tree/main/packages/ai) - pi-ai image content block shape (`{ type: "image", data, mimeType }`)

### Tertiary (LOW confidence)
- Stream-json input format for image content blocks -- undocumented whether content can be an array vs string. Flagged for validation if needed in future.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - all libraries already in use, no new dependencies
- Architecture: HIGH - incremental additions to existing well-understood code
- Pitfalls: HIGH - informed by official Node.js docs, Anthropic API docs, and documented real-world issues
- Image handling: MEDIUM - placeholder approach is straightforward and decided; actual image passthrough via stream-json is undocumented
- Timeout duration: MEDIUM - 180 seconds based on documented worst-case gaps, but real-world maximum is unknown

**Research date:** 2026-03-14
**Valid until:** 2026-04-14 (stable domain, Node.js subprocess APIs and Anthropic streaming protocol unlikely to change)
