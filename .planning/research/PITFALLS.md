# Domain Pitfalls

**Domain:** CLI subprocess integration bridging streaming protocols (Claude CLI to pi coding agent)
**Researched:** 2026-03-13

## Critical Pitfalls

Mistakes that cause rewrites, hangs, or render the extension unusable.

### Pitfall 1: Claude CLI Process Never Exits After Completion (stream-json hang)

**What goes wrong:** After the Claude CLI subprocess successfully completes a task and emits a `{"type":"result","subtype":"success"}` event, the process does not exit. stdout remains open, the process stays alive indefinitely, and the parent never receives an `end` event on the stream. This is a known, documented bug in Claude Code (issues #25629, #21099, #3187).

**Why it happens:** The CLI's stream-json mode does not close stdout or call `process.exit()` after sending the final result event. Internal timers, MCP servers, or pending promises keep the event loop alive. The process waits for more input on stdin even though the conversation is functionally complete.

**Consequences:** Every single request leaves an orphaned `claude` process consuming memory and CPU. Over a session, dozens of zombie processes accumulate. If using `for await` or `stream.on('end')` to detect completion, the handler never fires and the pi extension hangs forever.

**Prevention:**
- Parse the stdout NDJSON stream and detect the `{"type":"result"}` event as the authoritative "conversation done" signal.
- After receiving a result event, set a short grace period (e.g., 5 seconds) then forcefully kill the subprocess via `SIGKILL` (Windows: `taskkill /F /T /PID`).
- Never rely on the child process exiting cleanly or stdout closing. Always implement a kill-after-result pattern.
- Track all spawned child PIDs and implement cleanup on extension deactivation/disposal.

**Detection:** Monitor for growing process count (`claude` processes in task manager), or set a maximum wall-clock timeout per request and alert when it fires.

**Confidence:** HIGH -- documented in multiple GitHub issues with reproduction steps. The workaround (kill after result event) is recommended by issue reporters.

**Phase relevance:** Must be addressed in the very first phase (core subprocess management). Getting this wrong makes everything else untestable.

---

### Pitfall 2: NDJSON Lines Split Across Chunk Boundaries

**What goes wrong:** Node.js `child.stdout` emits `data` events as raw Buffer chunks with no guarantee of alignment to newline boundaries. A single NDJSON line may be split across two or more chunks, or a single chunk may contain multiple complete lines plus a partial trailing line. Naively calling `JSON.parse()` on each chunk produces `SyntaxError: Unexpected end of JSON input` intermittently and unpredictably.

**Why it happens:** OS pipe buffering delivers data in fixed-size blocks (typically 4KB-64KB depending on platform), not line-delimited units. Under load or with large tool outputs, this happens frequently.

**Consequences:** Intermittent JSON parse errors that corrupt the stream state. Missing events (tool calls, text deltas) that produce garbled output in pi. Extremely hard to reproduce in testing because it depends on timing and output size.

**Prevention:**
- Use a line-splitting transform stream like `split2` (npm package) that buffers partial lines and only emits complete newline-terminated strings.
- Pipe the child stdout through `split2` before any JSON parsing: `child.stdout.pipe(split2()).on('data', line => JSON.parse(line))`.
- Alternatively, implement a manual line buffer: accumulate chunks, split on `\n`, keep the last incomplete fragment for the next chunk.
- Wrap `JSON.parse()` in try/catch even after splitting -- malformed output from debug message leakage (see Pitfall 3) will still occur.

**Detection:** Add error logging around JSON.parse calls. If you see occasional parse errors that "fix themselves" on retry, you have this bug.

**Confidence:** HIGH -- this is a fundamental property of Node.js streams, well-documented across the ecosystem.

**Phase relevance:** Core subprocess communication layer (Phase 1). Must be correct before any event bridging can work.

---

### Pitfall 3: Debug/Diagnostic Output Corrupting stdout JSON Stream

**What goes wrong:** The Claude CLI binary sometimes writes non-JSON debug or diagnostic messages to stdout instead of stderr, even when `--output-format stream-json` and `--debug-to-stderr` flags are set. Examples include sandbox initialization messages like `[SandboxDebug] ...` appearing on stdout.

**Why it happens:** A documented bug in Claude Code (issue #12007, #14442) where internal components (sandbox initialization, native binary wrappers) write to stdout before the stream-json formatter takes control. The `--debug-to-stderr` flag is not universally respected by all internal subsystems.

**Consequences:** `JSON.parse()` throws on non-JSON lines, potentially crashing the extension or causing it to lose sync with the protocol state machine. This is particularly insidious because it may only happen on certain platforms (Linux with seccomp) or CLI versions.

**Prevention:**
- After line-splitting (Pitfall 2), validate that each line starts with `{` before attempting `JSON.parse()`.
- Skip/log non-JSON lines rather than crashing. Treat them as diagnostic noise.
- Always capture stderr separately and log it for debugging, but do not mix it into the JSON parse pipeline.
- Pin or document minimum Claude CLI versions and test across updates.

**Detection:** Search extension logs for "SyntaxError" or "Unexpected token" errors from the stdout parser. Any non-JSON line on stdout is a symptom.

**Confidence:** HIGH -- documented in multiple Claude Code GitHub issues with specific reproduction details.

**Phase relevance:** Core subprocess communication layer (Phase 1). The stdout parser must be resilient to this from day one.

---

### Pitfall 4: Windows-Specific Subprocess Spawning Failures

**What goes wrong:** On Windows, the `claude` command resolves to a `.cmd` batch file wrapper, not a native executable. `child_process.spawn('claude', [...args])` without `shell: true` fails silently or throws `ENOENT` because Windows cannot execute `.cmd` files directly. Additionally, stdin buffering on Windows differs from Unix -- data written to `child.stdin` may remain buffered in the Node.js layer and never reach the subprocess.

**Why it happens:** Windows does not support Unix-style shebangs or direct execution of batch scripts. The npm-installed `claude` is a `.cmd` wrapper. Windows pipes also use different buffering semantics than Unix pipes.

**Consequences:** Extension works on macOS/Linux during development but fails completely on Windows. If stdin buffering is the issue, the subprocess spawns but never receives the control_response, hanging on the first tool approval request. This exact bug was documented in the Claude Agent SDK Python (issue #208) where Windows stdin flush was missing.

**Prevention:**
- Always use `shell: true` in spawn options, or detect the platform and use `claude.cmd` on Windows explicitly.
- Better: use `cross-spawn` npm package which handles `.cmd` resolution transparently.
- After every `child.stdin.write()`, call `child.stdin.uncork()` or ensure the write completes by checking the return value and listening for `drain`.
- Test on Windows early and often. Do not treat it as a "we'll fix it later" platform.

**Detection:** Extension fails immediately on Windows with ENOENT or hangs during the first tool approval. Check `process.platform === 'win32'` code paths exist.

**Confidence:** HIGH -- documented in Claude SDK issues (#208, #252, #771) and Node.js documentation. The PROJECT.md explicitly lists Windows as a required platform.

**Phase relevance:** Must be addressed in Phase 1 (subprocess spawning). Cannot defer Windows support when the developer is on Windows.

---

### Pitfall 5: Bidirectional Stdin/Stdout Deadlock

**What goes wrong:** The Claude CLI sends a `control_request` on stdout asking for tool approval, then blocks waiting for a `control_response` on stdin. Meanwhile, the parent process is stuck trying to write data to stdin but the write buffer is full because the child's stdout buffer is also full and unread. Both processes wait on each other -- classic deadlock.

**Why it happens:** OS pipe buffers are finite (typically 64KB on Linux, 4KB on some Windows configurations). If the parent is not draining stdout fast enough (e.g., doing synchronous processing of each event before reading the next), the child's stdout buffer fills up. The child blocks on its next write. If the parent then tries to write to stdin, and the child is blocked so it cannot read stdin, both sides deadlock.

**Consequences:** The extension freezes mid-request with no error. The Claude subprocess is alive but blocked. No timeout fires because both processes are technically still running. Requires process kill to recover.

**Prevention:**
- Always read stdout asynchronously and buffer events in-memory. Never do synchronous processing that blocks the event loop while stdout data is arriving.
- Use Node.js streams properly -- pipe stdout through a transform, do not use synchronous `on('data')` handlers that block.
- Set `highWaterMark` on stdin to a reasonable size if writing large prompts.
- Implement a watchdog timer: if no stdout data arrives for N seconds after sending a control_response, assume deadlock and kill the process.

**Detection:** Extension hangs with no error output. Both parent and child processes show as running but idle in task manager. Adding verbose logging to stdin writes and stdout reads will show the last event before the hang.

**Confidence:** HIGH -- fundamental IPC deadlock pattern, well-documented in OS and Node.js literature. Especially relevant here because the control protocol requires synchronous request-response over the same pipe pair.

**Phase relevance:** Core subprocess communication (Phase 1). Must design the event loop correctly from the start.

---

### Pitfall 6: Control Protocol Timing -- Late or Missed control_response

**What goes wrong:** The Claude CLI sends a `control_request` with `subtype: "can_use_tool"` and expects a `control_response` with the matching `request_id` on stdin. If the response is not sent promptly, the CLI may time out, hang, or proceed with a default behavior (which may be to execute the tool -- the opposite of what the extension wants).

**Why it happens:** The extension needs to: (1) parse the NDJSON line from stdout, (2) determine whether to allow or deny the tool, (3) construct the response JSON, (4) write it to stdin with a trailing newline. If any step introduces latency (e.g., async lookup of tool mappings, awaiting a pi callback), the response arrives late. The `request_id` matching is also easy to get wrong if multiple control requests arrive in quick succession.

**Consequences:** Built-in tools execute inside the Claude CLI subprocess instead of being denied. This means file operations happen twice (once in Claude, once in pi), or worse, Claude writes files that pi does not know about, creating inconsistent state. For tool denial, this is a correctness-critical path.

**Prevention:**
- Pre-compute the allow/deny decision using a synchronous lookup (a Set of allowed tool name prefixes like `mcp__`).
- Write the control_response immediately upon parsing the control_request, before processing any other events. The denial decision is deterministic and does not need async operations.
- Always echo back the exact `request_id` from the request. Store nothing -- respond inline.
- Add logging for every control_request received and control_response sent, with timestamps, to detect latency.
- Implement a timeout: if a control_request is received but no response sent within 1 second, log an error.

**Detection:** Claude CLI executing tools that should be denied (visible in stream events as tool execution results appearing without pi having executed them). Grep extension logs for `control_request` without matching `control_response`.

**Confidence:** HIGH -- the control protocol is the core correctness mechanism for tool denial. The PROJECT.md explicitly calls out that CLI flags alone cannot achieve "propose but don't execute."

**Phase relevance:** Tool denial implementation (Phase 1-2). This is the defining architectural differentiator of the project.

---

## Moderate Pitfalls

### Pitfall 7: Token Explosion from Context Re-injection

**What goes wrong:** Each stateless subprocess invocation spawns a fresh `claude -p` process. The CLI automatically loads `CLAUDE.md`, MCP tool descriptions, plugin skills, and global settings on startup. When the extension replays full conversation history as a flattened prompt, the effective context per request balloons to 50K+ tokens before any actual work begins. Over a multi-turn session, cumulative token consumption becomes enormous.

**Why it happens:** The stateless model (spawn-per-request) means the CLI re-initializes everything each time. Combined with full history replay, you get O(n^2) token growth where n is conversation length -- turn 5 replays turns 1-4, turn 6 replays turns 1-5, etc.

**Prevention:**
- Use `--system-prompt` to inject only what the subagent actually needs, blocking the CLI's default context loading where possible.
- Consider `--no-user-config` or equivalent flags to prevent loading `~/.claude.json` settings.
- Implement conversation summarization for long sessions -- after N turns, summarize older turns rather than replaying verbatim.
- Monitor token counts in the result event and alert when they exceed thresholds.
- Document the tradeoff: stateless is simpler but more expensive. If token costs become prohibitive, persistent sessions may need to be reconsidered.

**Detection:** Result events include token usage data. Track tokens_used per request and alert on upward trends within a session.

**Confidence:** MEDIUM -- documented in the "50K tokens per subprocess turn" blog post and Claude Code issues. Severity depends on actual conversation lengths in pi workflows.

**Phase relevance:** Optimization phase (Phase 3+). The stateless model should work correctly first, then be optimized for token efficiency.

---

### Pitfall 8: Tool Name/Argument Mapping Mismatches

**What goes wrong:** Bidirectional tool mapping between Claude's built-in names (Read, Write, Edit, Bash, Grep, Glob) and pi's equivalents (read, write, edit, bash, grep, find) requires translating both tool names and their argument schemas. Missing a mapping or getting an argument translation wrong causes tools to fail silently, produce wrong results, or throw errors that surface as unhelpful messages to the user.

**Why it happens:** The argument schemas differ between systems: Claude uses `file_path` where pi uses `path`; Claude's Edit uses `old_string`/`new_string` where pi uses `oldText`/`newText`. The mapping is a manual, hand-maintained lookup table. New tools or argument changes in either system break the mapping silently.

**Consequences:** Tool calls that work in testing break when Claude uses unexpected argument combinations. Partial mapping (e.g., mapping the name but not all arguments) produces cryptic errors deep in pi's tool execution layer. Worst case: a Write tool call goes through with wrong arguments and corrupts a file.

**Prevention:**
- Define the mapping as a single, centralized, well-typed data structure (a Map or object with TypeScript interfaces for both sides).
- Write unit tests that verify every mapped tool name has corresponding argument translations in both directions.
- Validate mapped arguments before passing to pi -- check required fields are present.
- Log the raw Claude tool call AND the translated pi tool call side-by-side for debugging.
- Watch Claude Code changelogs for tool schema changes.

**Detection:** Tool execution errors from pi that mention missing required arguments. Diff the raw Claude tool_use event against the translated pi tool call in logs.

**Confidence:** HIGH -- the reference project (claude-agent-sdk-pi) maintains this mapping, confirming it is necessary and error-prone.

**Phase relevance:** Tool bridging implementation (Phase 2). Must be comprehensive and tested.

---

### Pitfall 9: Stream Event Translation Gaps and Ordering Errors

**What goes wrong:** Claude's API stream events (content_block_start, content_block_delta, content_block_stop, message_start, message_delta, message_stop) must be mapped to pi's event format (text_start, text_delta, text_end, toolcall_start, toolcall_delta, toolcall_end, thinking_start, thinking_delta, thinking_end, done, error). Missing an event, emitting events out of order, or failing to handle edge cases (e.g., empty content blocks, multiple tool calls in a single message) produces broken output in pi.

**Why it happens:** The two event protocols have different semantics. Claude uses `content_block_start` with an `index` field for multiplexing multiple blocks; pi expects sequential start/delta/end triplets. Claude may emit multiple content blocks in parallel (e.g., a thinking block and a text block simultaneously). Claude's `message_delta` contains stop_reason; pi's `done` event is separate.

**Consequences:** Text appears jumbled in pi's UI. Tool calls show incomplete arguments. Thinking blocks are missing or interleaved with text. The `done` event fires before all text is delivered, causing truncation.

**Prevention:**
- Build a state machine that tracks the current "active block" by index and type (text, tool_use, thinking).
- Emit pi events only on state transitions (e.g., emit `text_end` when a new block starts or the message ends, not on every delta).
- Handle the edge case where Claude sends `content_block_stop` for one block and `content_block_start` for another in the same chunk.
- Write integration tests that replay captured NDJSON streams from real Claude sessions and verify the pi event sequence.
- Pay special attention to `message_delta` with `stop_reason: "end_turn"` vs `stop_reason: "tool_use"` -- they require different pi event sequences.

**Detection:** Compare pi's displayed output against the raw Claude stream events. Missing text or garbled tool calls indicate mapping bugs.

**Confidence:** HIGH -- the reference project's core complexity is in this translation layer, confirming it is non-trivial.

**Phase relevance:** Stream bridging implementation (Phase 2). Requires careful state management.

---

### Pitfall 10: MCP Proxy Server Lifecycle Management

**What goes wrong:** Custom pi tools need to be exposed to the Claude CLI via an MCP server. The reference project uses the SDK's `createSdkMcpServer()`. This project must implement its own MCP server (likely stdio-based) spawned alongside the Claude process and registered via `--mcp-config`. If the MCP server starts too slowly, crashes, or is not properly cleaned up, the Claude process either fails to discover the tools or hangs during initialization.

**Why it happens:** The MCP SDK has a 60-second default timeout for server connections. If the MCP server takes longer to start (e.g., waiting for pi to register tools), the connection fails. On the other end, if the MCP server outlives the Claude subprocess, it becomes an orphaned process.

**Consequences:** Custom pi tools are unavailable to Claude, silently degrading capability. Or the MCP server process leaks on every request, accumulating like the Claude subprocess hang (Pitfall 1). In the worst case, a crashing MCP server causes the Claude process to error out mid-conversation.

**Prevention:**
- Start the MCP server before spawning the Claude process. Verify it is ready (listening) before passing it via `--mcp-config`.
- Use stdio transport for the MCP server (not HTTP) to simplify lifecycle -- the server dies when its stdin/stdout pipes close.
- Tie the MCP server's lifecycle to the Claude subprocess's lifecycle. When the Claude process is killed (Pitfall 1), ensure the MCP server is also killed.
- Implement health checks: if the MCP server stops responding, kill and restart both it and the Claude process.
- Consider an in-process MCP server (using Node.js worker threads or direct function calls) to avoid process management complexity entirely.

**Detection:** Claude's stream events will not contain any `mcp__custom-tools__*` tool calls if the MCP server failed to connect. Log MCP server startup and connection events.

**Confidence:** MEDIUM -- the reference project sidesteps this by using the SDK's built-in MCP server. This project must solve it independently.

**Phase relevance:** Custom tool proxy (Phase 3). Can be deferred until built-in tool mapping works.

---

### Pitfall 11: Subprocess Startup Latency per Request

**What goes wrong:** Each pi LLM request spawns a fresh `claude -p` subprocess. The Claude CLI has ~12 seconds of startup overhead (module loading, environment setup, config reading, MCP server initialization). This means every single tool-using interaction in pi takes 12+ seconds before the LLM even starts generating.

**Why it happens:** The stateless model requires a fresh process per request. The Claude CLI is not designed for rapid start-stop cycles -- it is designed for interactive sessions.

**Consequences:** The extension feels unacceptably slow compared to direct API-based pi providers. Users abandon it after a few interactions. Multi-step agentic workflows that involve many LLM calls become unusable.

**Prevention:**
- Accept the latency for the initial implementation (Phase 1). Do not prematurely optimize.
- Profile the actual startup time with the specific CLI flags used (`-p --input-format stream-json --output-format stream-json --verbose`). The 12-second figure is from SDK benchmarks and may differ.
- For Phase 3+, investigate persistent subprocess sessions (stream-json multi-turn mode) where the process stays alive between requests. The PROJECT.md lists this as "Out of Scope" but it may become necessary.
- Consider a process warm-up strategy: pre-spawn a Claude process during idle time so it is ready when the next request arrives.

**Detection:** Measure time from request receipt to first token emission. If consistently >10 seconds, startup latency dominates.

**Confidence:** HIGH -- documented in Claude Agent SDK issue #34 with benchmarks.

**Phase relevance:** All phases, but optimization in Phase 3+. Correctness before performance.

---

## Minor Pitfalls

### Pitfall 12: Forgetting to End stdin After Prompt Delivery (Stateless Mode)

**What goes wrong:** In stateless `-p` mode, the full prompt is sent once and then the CLI processes it. If `child.stdin.end()` is not called after writing the prompt (or the stream-json initial message), the CLI may wait for more input instead of processing what it has.

**Prevention:** Call `child.stdin.end()` after writing the last NDJSON message for the initial prompt. In stream-json mode where you need stdin open for control_responses, do NOT end stdin -- but be aware that this means the CLI's "is input complete?" signal must come from the message content, not the stream closing. Verify which mode you are in and handle accordingly.

**Confidence:** MEDIUM -- depends on how stream-json mode handles input completion vs the simpler `-p` pipe mode.

**Phase relevance:** Phase 1 (subprocess communication).

---

### Pitfall 13: Model ID Mismatch Between pi and Claude CLI

**What goes wrong:** pi's `getModels("anthropic")` returns model IDs in one format (e.g., `claude-sonnet-4-5-20250929`). The Claude CLI may expect model IDs in a different format, or the CLI may not support passing model IDs at all when using Pro/Max subscription auth (the subscription determines the model tier).

**Prevention:** Test which `--model` flag values the CLI accepts with subscription auth. Map pi model IDs to CLI-accepted values. If the CLI ignores `--model` with subscription auth, document this limitation and default to the subscription's model.

**Confidence:** LOW -- needs experimental verification.

**Phase relevance:** Phase 1 (subprocess spawning arguments).

---

### Pitfall 14: stderr Noise Interpreted as Errors

**What goes wrong:** The Claude CLI writes progress messages, warnings, and diagnostic information to stderr. The extension captures stderr and interprets any output as an error condition, triggering error events in pi or aborting the request.

**Prevention:** Capture stderr and log it, but do not treat stderr output as an error condition. Only treat non-zero exit codes (combined with no result event) as errors. The `result` event's `subtype` field (`success` vs `error`) is the authoritative error signal.

**Confidence:** HIGH -- standard subprocess best practice.

**Phase relevance:** Phase 1 (subprocess management).

---

### Pitfall 15: Race Between Process Exit and Final stdout Data

**What goes wrong:** The Claude subprocess exits (emits `close` event) before all stdout data has been consumed by the parent. The remaining buffered data is lost, potentially including the final `result` event.

**Prevention:** Always process the `close` event after the stdout `end` event. Use `child.on('close')` not `child.on('exit')` as the authoritative "process is done" signal, since `close` fires after all stdio streams are closed. Even better, rely on the `result` event from the NDJSON stream (Pitfall 1) rather than process lifecycle events.

**Confidence:** HIGH -- well-documented Node.js child_process behavior.

**Phase relevance:** Phase 1 (subprocess lifecycle).

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Subprocess spawning (Phase 1) | #1 Process never exits, #4 Windows spawn failure, #5 Deadlock | Kill-after-result, cross-spawn, async stdout reading |
| NDJSON parsing (Phase 1) | #2 Split chunks, #3 Debug corruption | split2 + JSON validation per line |
| Control protocol (Phase 1-2) | #6 Late control_response | Synchronous allow/deny decision, immediate write |
| Tool mapping (Phase 2) | #8 Argument mismatch | Typed centralized mapping, unit tests |
| Stream event bridging (Phase 2) | #9 Event ordering | State machine, integration tests with captured streams |
| MCP proxy (Phase 3) | #10 Lifecycle leaks | Stdio transport, tied lifecycle, in-process option |
| Performance (Phase 3+) | #7 Token explosion, #11 Startup latency | System prompt isolation, potential persistent sessions |

## Sources

- [Claude Code CLI hangs after result event - Issue #25629](https://github.com/anthropics/claude-code/issues/25629)
- [Claude Code stream-json input hang - Issue #3187](https://github.com/anthropics/claude-code/issues/3187)
- [Claude Code debug messages corrupt stdout - Issue #12007](https://github.com/anthropics/claude-code/issues/12007)
- [Claude Code persistent JSON parse error on Windows - Issue #14442](https://github.com/anthropics/claude-code/issues/14442)
- [Claude Code can't be spawned from Node.js - Issue #771](https://github.com/anthropics/claude-code/issues/771)
- [ClaudeSDKClient hangs on Windows - Issue #208](https://github.com/anthropics/claude-agent-sdk-python/issues/208)
- [Claude SDK Windows path detection - Issue #252](https://github.com/anthropics/claude-agent-sdk-python/issues/252)
- [Claude Agent SDK 12s overhead per call - Issue #34](https://github.com/anthropics/claude-agent-sdk-typescript/issues/34)
- [50K tokens per subprocess turn - DEV Community](https://dev.to/jungjaehoon/why-claude-code-subagents-waste-50k-tokens-per-turn-and-how-to-fix-it-41ma)
- [Node.js Child Process Documentation](https://nodejs.org/api/child_process.html)
- [Node.js Backpressuring in Streams](https://nodejs.org/en/learn/modules/backpressuring-in-streams)
- [split2 npm package](https://www.npmjs.com/package/split2)
- [Node.js child_process stdout truncation - Issue #19218](https://github.com/nodejs/node/issues/19218)
- [Node.js spawn stdin issues - Issue #2985](https://github.com/nodejs/node/issues/2985)
- [Claude Code duplicate session entries - Issue #5034](https://github.com/anthropics/claude-code/issues/5034)
- [claude-agent-sdk-pi reference project](https://github.com/prateekmedia/claude-agent-sdk-pi)
