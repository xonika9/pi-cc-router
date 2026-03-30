# Phase 1: Core Subprocess Bridge - Research

**Researched:** 2026-03-13
**Domain:** Node.js subprocess management, NDJSON stream parsing, pi custom provider API, Claude CLI stream-json protocol
**Confidence:** HIGH

## Summary

Phase 1 establishes the foundational bridge between pi's custom provider interface and the Claude Code CLI subprocess. The scope is deliberately narrow: register as a provider, spawn a `claude -p` subprocess with stream-json flags, parse NDJSON stdout, and bridge **text-only** streaming events back to pi. Tool handling, thinking blocks, usage metrics, and MCP proxy are explicitly out of scope for this phase.

The reference implementation (`claude-agent-sdk-pi`) provides a proven, complete pattern for provider registration, model derivation, history serialization, and event bridging. This phase replicates the same architecture but replaces the Agent SDK's `query()` transport with direct `child_process.spawn()` + readline-based NDJSON parsing. The key technical risks are subprocess lifecycle management (the CLI hangs after completion -- a known bug requiring force-kill), NDJSON parsing resilience (debug output corrupting stdout), and Windows `.cmd` shim resolution (requires `cross-spawn`).

**Primary recommendation:** Follow the reference project's patterns exactly for provider registration and event bridging. Use `cross-spawn` + `readline` for subprocess I/O. Implement kill-after-result as a mandatory subprocess cleanup pattern from day one.

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions

- Provider ID: `pi-claude-cli` (follows reference project convention of using the package name)
- Models: derived dynamically from `getModels("anthropic")` -- auto-updates when pi adds new models
- Display names: unchanged from Anthropic catalog (no suffix)
- `api` field: `"pi-claude-cli"` (matches provider concept)
- `baseUrl` and `apiKey`: placeholder strings (not used for actual HTTP/auth)
- History: flatten entire conversation history into a single text prompt with USER:/ASSISTANT:/TOOL RESULT: labels
- Send as a single user message to the subprocess
- System prompt: pass pi's system context through to Claude via `--append-system-prompt` or equivalent
- Also append AGENTS.md content and skill definitions from pi config (matching reference project behavior)
- Sanitize path references (`.pi` -> `.claude`) for Claude Code compatibility
- CLI flags: `--no-session-persistence`, `--model`, `--permission-mode dontAsk`, `--input-format stream-json`, `--output-format stream-json`, `--verbose`
- Startup validation: check CLI presence on PATH at provider registration time; check auth status; fail gracefully with clear error messages if either fails

### Claude's Discretion

- NDJSON parsing error handling details (malformed lines, partial JSON)
- Exact subprocess spawn options (stdio configuration, environment variables)
- Stream event buffering strategy
- Internal error message wording

### Deferred Ideas (OUT OF SCOPE)

None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>

## Phase Requirements

| ID      | Description                                                                                                        | Research Support                                                                                                                                                                           |
| ------- | ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| PROV-01 | Register as custom pi provider via `pi.registerProvider()` with unique provider ID                                 | Reference project provides exact pattern: `pi.registerProvider("pi-claude-cli", { ... })` with `streamSimple` handler. Models from `getModels("anthropic")`.                               |
| PROV-02 | Expose all current Claude models with correct context windows, max tokens, and cost info                           | `getModels("anthropic")` returns complete model catalog with all required fields. Reference project maps `.map()` to copy id, name, reasoning, input, cost, contextWindow, maxTokens.      |
| PROV-03 | Implement `streamSimple` handler as core entry point                                                               | Reference project's `streamClaudeAgentSdk` function is the template. Returns `createAssistantMessageEventStream()`. Runs async IIFE internally.                                            |
| PROC-01 | Spawn `claude -p --input-format stream-json --output-format stream-json --verbose` as fresh subprocess per request | Use `cross-spawn` with `stdio: ["pipe", "pipe", "pipe"]`. Write user message to stdin as NDJSON. Include `--include-partial-messages` for token-by-token streaming.                        |
| STRM-01 | Parse NDJSON output line-by-line via `readline`, validating each line as JSON                                      | `readline.createInterface({ input: proc.stdout, crlfDelay: Infinity })` with `for await` loop. Guard each line with `line.startsWith("{")` before `JSON.parse()`.                          |
| STRM-02 | Bridge Claude API stream events to pi's `AssistantMessageEventStream` events                                       | State machine tracking content block index and type. For Phase 1: only bridge `text` content blocks (text_start/delta/end). Tool and thinking blocks deferred to later phases.             |
| HIST-01 | Build flattened prompt from full pi conversation history with role labeling                                        | Reference project's `buildPromptBlocks()` is the template. Labels: `USER:`, `ASSISTANT:`, `TOOL RESULT (historical <name>):`. For Phase 1, text-only content (images deferred to Phase 5). |

</phase_requirements>

## Standard Stack

### Core

| Library              | Version             | Purpose                            | Why Standard                                                                                                                                                    |
| -------------------- | ------------------- | ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `cross-spawn`        | ^7.0.6              | Cross-platform subprocess spawning | Handles Windows `.cmd` shim resolution transparently. Drop-in replacement for `child_process.spawn()`. Used by virtually every cross-platform Node.js CLI tool. |
| `node:child_process` | Built-in (Node 22+) | Subprocess spawn API               | Native Node.js API. `cross-spawn` wraps this.                                                                                                                   |
| `node:readline`      | Built-in (Node 22+) | Line-by-line stdout parsing        | Splits NDJSON stream into complete lines. Handles partial line buffering and `\n` boundaries correctly. No external dependency needed.                          |

### Supporting

| Library                         | Version            | Purpose                                                                  | When to Use                                                      |
| ------------------------------- | ------------------ | ------------------------------------------------------------------------ | ---------------------------------------------------------------- |
| `@mariozechner/pi-ai`           | ^0.52.0 (peer dep) | `createAssistantMessageEventStream`, `getModels`, `calculateCost`, types | Always -- provides the pi event stream factory and model catalog |
| `@mariozechner/pi-coding-agent` | ^0.52.0 (peer dep) | `ExtensionAPI` type, `AuthStorage`                                       | Always -- provides the extension entry point interface           |

### NOT Needed in Phase 1

| Library                     | Purpose                              | Phase   |
| --------------------------- | ------------------------------------ | ------- |
| `@modelcontextprotocol/sdk` | MCP server for custom tool proxy     | Phase 4 |
| `zod`                       | Schema validation (MCP SDK peer dep) | Phase 4 |

### Alternatives Considered

| Instead of    | Could Use                         | Tradeoff                                                                                                                                      |
| ------------- | --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `cross-spawn` | Native `spawn` with `shell: true` | `shell: true` introduces command injection risk and changes argument quoting. `cross-spawn` solves Windows `.cmd` without security tradeoffs. |
| `readline`    | `split2` npm package              | `split2` works but adds unnecessary dependency. `readline` is built-in and handles the same buffering correctly.                              |
| `readline`    | Manual `data` event buffering     | Error-prone. Chunks don't align to newline boundaries (Pitfall 2). `readline` handles this correctly.                                         |

**Installation (Phase 1 only):**

```bash
npm install cross-spawn
npm install --save-dev typescript @types/node
```

## Architecture Patterns

### Recommended Project Structure

```
pi-claude-cli/
  index.ts            # Extension entry point (or extensions/index.ts)
  src/
    provider.ts       # registerProvider + streamSimple orchestration
    prompt-builder.ts # History flattening (USER:/ASSISTANT:/TOOL RESULT:)
    process-manager.ts # Subprocess spawn, stdin write, lifecycle
    stream-parser.ts  # NDJSON line parsing + message classification
    event-bridge.ts   # Claude API events -> pi AssistantMessageEventStream
    types.ts          # Shared type definitions for wire protocol messages
  package.json
  tsconfig.json
```

**Note:** Pi loads `.ts` files directly via `jiti` -- no build step needed for distribution. The `tsconfig.json` is for IDE type checking and `tsc --noEmit` only.

### Pattern 1: Extension Entry Point

**What:** Default export function receiving `ExtensionAPI`, registers provider with models and `streamSimple` handler.
**When to use:** Always -- this is the pi extension contract.
**Example (derived from reference project):**

```typescript
// Source: claude-agent-sdk-pi/index.ts (adapted)
import {
  getModels,
  type AssistantMessageEventStream,
  type Context,
  type Model,
  type SimpleStreamOptions,
} from "@mariozechner/pi-ai";
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";

const PROVIDER_ID = "pi-claude-cli";

const MODELS = getModels("anthropic").map((model) => ({
  id: model.id,
  name: model.name,
  reasoning: model.reasoning,
  input: model.input,
  cost: model.cost,
  contextWindow: model.contextWindow,
  maxTokens: model.maxTokens,
}));

export default function (pi: ExtensionAPI) {
  // Startup validation
  validateCliPresence(); // throws with install instructions if missing
  validateCliAuth(); // warns if not authenticated

  pi.registerProvider(PROVIDER_ID, {
    baseUrl: "pi-claude-cli",
    apiKey: "unused",
    api: "pi-claude-cli",
    models: MODELS,
    streamSimple: (model, context, options) => {
      return streamViaCli(model, context, options);
    },
  });
}
```

### Pattern 2: Stateless Subprocess per Request

**What:** Each `streamSimple` call spawns a fresh `claude -p` process, sends full conversation as a single user message, and bridges streaming events back.
**When to use:** Every LLM turn.
**Why:** Matches reference project pattern. Avoids session state complexity. Proven approach.
**Example:**

```typescript
// Source: Architecture research + reference project pattern
import { createAssistantMessageEventStream } from "@mariozechner/pi-ai";
import spawn from "cross-spawn";

function streamViaCli(model, context, options?): AssistantMessageEventStream {
  const stream = createAssistantMessageEventStream();

  (async () => {
    try {
      const prompt = buildPrompt(context);
      const proc = spawnClaude(model, options);

      // Write user message to stdin as NDJSON
      const userMessage = JSON.stringify({
        type: "user",
        message: { role: "user", content: prompt },
      });
      proc.stdin.write(userMessage + "\n");
      // Do NOT close stdin -- needed for control_response in later phases

      // Parse and bridge events
      await parseAndBridgeEvents(proc, stream, model, options);
    } catch (error) {
      stream.push({ type: "error", reason: "error", error: output });
      stream.end();
    }
  })();

  return stream;
}
```

### Pattern 3: Kill-After-Result Subprocess Cleanup

**What:** Detect the `{"type":"result"}` NDJSON message as the authoritative completion signal, then force-kill the subprocess after a short grace period.
**When to use:** Every subprocess -- the CLI hangs after completion (known bug).
**Why:** Without this, every request leaves an orphaned `claude` process. This is a critical correctness pattern.
**Example:**

```typescript
// Source: Pitfalls research, Claude Code issues #25629, #21099
function cleanupProcess(proc: ChildProcess): void {
  // Grace period for any final stdout flush
  setTimeout(() => {
    if (!proc.killed) {
      proc.kill("SIGKILL");
    }
  }, 2000);
}

// In the NDJSON parse loop:
for await (const line of rl) {
  const msg = safeParseLine(line);
  if (!msg) continue;

  if (msg.type === "result") {
    // Handle result (success or error)
    handleResult(msg, stream, output);
    cleanupProcess(proc);
    break; // Stop reading -- process will be killed
  }
  // ... handle other message types
}
```

### Pattern 4: NDJSON Resilient Parsing

**What:** Parse stdout line-by-line with `readline`, skip non-JSON lines (debug noise), wrap `JSON.parse` in try/catch.
**When to use:** All subprocess communication.
**Why:** Claude CLI sometimes writes non-JSON debug output to stdout (known bug). Readline handles partial line buffering.
**Example:**

```typescript
// Source: Pitfalls #2, #3; Node.js readline docs
import { createInterface } from "node:readline";

const rl = createInterface({
  input: proc.stdout,
  crlfDelay: Infinity,
  terminal: false,
});

for await (const line of rl) {
  const trimmed = line.trim();
  if (!trimmed || !trimmed.startsWith("{")) {
    // Skip empty lines and non-JSON debug output
    continue;
  }
  try {
    const msg = JSON.parse(trimmed);
    // Route by msg.type: "stream_event", "result", "system", etc.
  } catch (e) {
    // Log and skip malformed JSON -- do not crash
    console.error("Failed to parse NDJSON line:", trimmed);
  }
}
```

### Pattern 5: Event Bridge State Machine

**What:** Track content block state (index, type) to correctly translate Claude API streaming events to pi events.
**When to use:** Translating `stream_event` messages.
**Why:** Claude uses indexed content blocks; pi expects sequential start/delta/end groups.
**Example (Phase 1 -- text only):**

```typescript
// Source: Reference project streamClaudeAgentSdk function
// Phase 1: only handle text content blocks
if (event.type === "content_block_start") {
  if (event.content_block?.type === "text") {
    const block = { type: "text" as const, text: "", index: event.index };
    output.content.push(block);
    stream.push({
      type: "text_start",
      contentIndex: output.content.length - 1,
      partial: output,
    });
  }
  // tool_use and thinking: log warning, skip (Phase 2/3)
}

if (event.type === "content_block_delta") {
  if (event.delta?.type === "text_delta") {
    const idx = blocks.findIndex((b) => b.index === event.index);
    const block = blocks[idx];
    if (block?.type === "text") {
      block.text += event.delta.text;
      stream.push({
        type: "text_delta",
        contentIndex: idx,
        delta: event.delta.text,
        partial: output,
      });
    }
  }
}

if (event.type === "content_block_stop") {
  const idx = blocks.findIndex((b) => b.index === event.index);
  const block = blocks[idx];
  if (block?.type === "text") {
    delete (block as any).index;
    stream.push({
      type: "text_end",
      contentIndex: idx,
      content: block.text,
      partial: output,
    });
  }
}
```

### Pattern 6: Startup Validation

**What:** At provider registration time, verify `claude` CLI is on PATH and authenticated. Fail gracefully if not.
**When to use:** Extension activation (the default export function).
**Example:**

```typescript
// Source: CONTEXT.md locked decision
import { execSync } from "node:child_process";

function validateCliPresence(): void {
  try {
    execSync("claude --version", { stdio: "pipe", timeout: 5000 });
  } catch {
    throw new Error(
      "Claude Code CLI not found. Install it: npm install -g @anthropic-ai/claude-code\n" +
        "Then authenticate: claude auth login",
    );
  }
}

function validateCliAuth(): boolean {
  try {
    // claude auth status exits 0 if logged in, 1 if not
    execSync("claude auth status", { stdio: "pipe", timeout: 5000 });
    return true;
  } catch {
    console.warn(
      "[pi-claude-cli] Claude CLI is not authenticated. " +
        "Run 'claude auth login' to authenticate.",
    );
    return false;
  }
}
```

### Anti-Patterns to Avoid

- **Persistent subprocess sessions:** Adds complexity, unproven with stream-json, defer to v2. Each request is a fresh process.
- **Parsing stderr as protocol data:** stderr is diagnostic output. Only parse stdout as NDJSON.
- **Synchronous subprocess I/O:** Blocks event loop, prevents streaming. Always use async spawn with piped stdio.
- **Closing stdin after prompt:** In stream-json mode, stdin must stay open for `control_response` messages (needed in Phase 2+). Write the user message but do NOT call `stdin.end()`.
- **Relying on process exit for completion:** The CLI hangs after result (known bug). Use the `result` NDJSON message as the completion signal, then force-kill.

## Don't Hand-Roll

| Problem                       | Don't Build                                                   | Use Instead                                                      | Why                                                                                                                                        |
| ----------------------------- | ------------------------------------------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Windows `.cmd` resolution     | Platform detection + `shell: true` or manual `.cmd` extension | `cross-spawn`                                                    | Handles PATH resolution, `.cmd` wrappers, and argument quoting correctly across all platforms. Avoids `shell: true` security implications. |
| Line-by-line stream splitting | Manual `data` event buffering with `\n` splitting             | `readline.createInterface()`                                     | Handles partial line buffering, `\r\n` vs `\n`, and edge cases correctly. Built-in, zero dependencies.                                     |
| Model catalog                 | Hardcoded model list with manual version bumps                | `getModels("anthropic")` from `@mariozechner/pi-ai`              | Automatically stays current as pi updates its Anthropic model catalog. No maintenance burden.                                              |
| Event stream creation         | Custom EventEmitter or Observable                             | `createAssistantMessageEventStream()` from `@mariozechner/pi-ai` | Returns the exact type pi expects. Handles backpressure, end signaling, and type safety.                                                   |
| Cost calculation              | Manual token price math                                       | `calculateCost(model, usage)` from `@mariozechner/pi-ai`         | Uses the model's cost rates. Handles cache token pricing correctly.                                                                        |

**Key insight:** The pi runtime already provides model catalogs, event stream factories, and cost calculation. The reference project demonstrates using all of these. The only custom code needed is the subprocess bridge layer.

## Common Pitfalls

### Pitfall 1: CLI Process Never Exits (CRITICAL)

**What goes wrong:** After emitting `{"type":"result","subtype":"success"}`, the Claude CLI subprocess does not exit. The process hangs indefinitely, leaking resources.
**Why it happens:** Known CLI bug (issues #25629, #21099, #3187). Internal timers, MCP servers, or pending promises keep the event loop alive.
**How to avoid:** Detect the `result` NDJSON message as the completion signal. After a 2-second grace period, force-kill via `SIGKILL`. Never rely on process exit or stdout close.
**Warning signs:** Growing process count in task manager. `for await` loop never terminates.
**Confidence:** HIGH -- documented in multiple Claude Code GitHub issues.

### Pitfall 2: Debug Output Corrupting stdout JSON Stream

**What goes wrong:** Non-JSON diagnostic messages (e.g., `[SandboxDebug] ...`) appear on stdout before the stream-json formatter takes control.
**Why it happens:** CLI bug (issues #12007, #14442). Internal components write to stdout before structured output mode activates.
**How to avoid:** Validate each line starts with `{` before `JSON.parse()`. Skip/log non-JSON lines. Never crash on parse errors.
**Warning signs:** Intermittent `SyntaxError: Unexpected token` errors from the stdout parser.
**Confidence:** HIGH -- documented in Claude Code GitHub issues.

### Pitfall 3: Windows Subprocess Spawning Failure

**What goes wrong:** `spawn("claude", [...])` fails with ENOENT on Windows because `claude` resolves to a `.cmd` batch file.
**Why it happens:** Windows cannot execute `.cmd` files directly via `spawn()` without `shell: true`.
**How to avoid:** Use `cross-spawn` (the locked decision). It handles `.cmd` resolution transparently.
**Warning signs:** Extension works on macOS/Linux but throws ENOENT on Windows.
**Confidence:** HIGH -- fundamental Windows/Node.js behavior.

### Pitfall 4: Stdin Must Stay Open

**What goes wrong:** Calling `proc.stdin.end()` after writing the user message prevents sending `control_response` messages later.
**Why it happens:** Stream-json input mode uses stdin for both the initial prompt AND subsequent control responses. Closing stdin makes the subprocess unable to receive responses.
**How to avoid:** Write the user message but do NOT call `stdin.end()`. The subprocess knows the user message is complete from the NDJSON message structure, not from stdin closing.
**Warning signs:** First request works but tool approval hangs in Phase 2.
**Confidence:** HIGH -- inherent to the stream-json bidirectional protocol.

### Pitfall 5: Race Between Process Close and Final stdout Data

**What goes wrong:** The subprocess `close` event fires before all stdout data has been consumed.
**Why it happens:** Node.js `close` event can fire before stdout buffer is fully drained.
**How to avoid:** Do not rely on `close` or `exit` events. Use the `result` NDJSON message as the completion signal (which is what Pitfall 1's kill-after-result pattern already does).
**Warning signs:** Truncated output, missing final text or result event.
**Confidence:** HIGH -- documented Node.js child_process behavior.

### Pitfall 6: Model ID Format Mismatch

**What goes wrong:** `getModels("anthropic")` returns model IDs that the CLI's `--model` flag doesn't accept.
**Why it happens:** Pi model IDs (e.g., `claude-sonnet-4-5-20250929`) may differ from what the CLI expects.
**How to avoid:** Test `--model` flag values experimentally. The reference project passes model IDs directly and it works. Log the exact model ID being passed for debugging.
**Warning signs:** CLI errors about invalid model ID.
**Confidence:** LOW -- needs experimental verification. The reference project passes them directly and works.

## Code Examples

### Complete AssistantMessage Output Structure

```typescript
// Source: reference project streamClaudeAgentSdk function
const output: AssistantMessage = {
  role: "assistant",
  content: [], // Populated with { type: "text", text: string } blocks
  api: model.api, // "pi-claude-cli"
  provider: model.provider,
  model: model.id,
  usage: {
    input: 0,
    output: 0,
    cacheRead: 0,
    cacheWrite: 0,
    totalTokens: 0,
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0, total: 0 },
  },
  stopReason: "stop", // "stop" | "length" | "toolUse"
  timestamp: Date.now(),
};
```

### Complete Subprocess Spawn

```typescript
// Source: CONTEXT.md locked decisions + STACK.md recommendations
import spawn from "cross-spawn";

function spawnClaude(
  model: Model<any>,
  options?: SimpleStreamOptions,
): ChildProcess {
  const args = [
    "-p",
    "--input-format",
    "stream-json",
    "--output-format",
    "stream-json",
    "--verbose",
    "--include-partial-messages",
    "--no-session-persistence",
    "--model",
    model.id,
    "--permission-mode",
    "dontAsk",
  ];

  // System prompt handling
  if (systemPromptAppend) {
    args.push("--append-system-prompt", systemPromptAppend);
  }

  const proc = spawn("claude", args, {
    stdio: ["pipe", "pipe", "pipe"],
    cwd: options?.cwd ?? process.cwd(),
  });

  return proc;
}
```

### Complete History Flattening (Prompt Builder)

```typescript
// Source: reference project buildPromptBlocks (simplified for Phase 1 text-only)
function buildPrompt(context: Context): string {
  const parts: string[] = [];

  for (const message of context.messages) {
    if (message.role === "user") {
      parts.push("USER:");
      if (typeof message.content === "string") {
        parts.push(message.content);
      } else if (Array.isArray(message.content)) {
        for (const block of message.content) {
          if (block.type === "text") parts.push(block.text ?? "");
          // Images deferred to Phase 5 (HIST-02)
        }
      }
    } else if (message.role === "assistant") {
      parts.push("ASSISTANT:");
      parts.push(contentToText(message.content));
    } else if (message.role === "toolResult") {
      parts.push(`TOOL RESULT (historical ${message.toolName}):`);
      if (typeof message.content === "string") {
        parts.push(message.content);
      } else if (Array.isArray(message.content)) {
        for (const block of message.content) {
          if (block.type === "text") parts.push(block.text ?? "");
        }
      }
    }
  }

  return parts.join("\n") || "";
}

function contentToText(content: string | Array<any>): string {
  if (typeof content === "string") return content;
  if (!Array.isArray(content)) return "";
  return content
    .map((block) => {
      if (block.type === "text") return block.text ?? "";
      if (block.type === "thinking") return block.thinking ?? "";
      if (block.type === "toolCall") {
        const args = block.arguments ? JSON.stringify(block.arguments) : "{}";
        return `Historical tool call (non-executable): ${block.name} args=${args}`;
      }
      return `[${block.type}]`;
    })
    .join("\n");
}
```

### Complete User Message NDJSON Format

```typescript
// Source: Claude Agent SDK wire protocol (buildwithaws.substack.com)
// Write to subprocess stdin:
const userMessage = {
  type: "user",
  message: {
    role: "user",
    content: promptText, // The flattened history string
  },
};
proc.stdin.write(JSON.stringify(userMessage) + "\n");
// Do NOT call proc.stdin.end() -- stdin stays open for control_responses
```

### Complete Stream Event Message Format (what stdout produces)

```jsonl
{"type":"system","subtype":"init","session_id":"...","tools":[...]}
{"type":"stream_event","event":{"type":"message_start","message":{"id":"...","type":"message","role":"assistant","content":[],"model":"claude-sonnet-4-5-20250929","usage":{"input_tokens":100,"output_tokens":0}}}}
{"type":"stream_event","event":{"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}}
{"type":"stream_event","event":{"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Hello"}}}
{"type":"stream_event","event":{"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":" world"}}}
{"type":"stream_event","event":{"type":"content_block_stop","index":0}}
{"type":"stream_event","event":{"type":"message_delta","delta":{"stop_reason":"end_turn"},"usage":{"output_tokens":5}}}
{"type":"stream_event","event":{"type":"message_stop"}}
{"type":"result","subtype":"success","result":"Hello world","session_id":"..."}
```

### Abort Signal Handling

```typescript
// Source: reference project abort handling
const onAbort = () => {
  wasAborted = true;
  if (!proc.killed) {
    proc.kill("SIGTERM");
  }
};

if (options?.signal) {
  if (options.signal.aborted) {
    onAbort();
  } else {
    options.signal.addEventListener("abort", onAbort, { once: true });
  }
}

// In finally block:
if (options?.signal) {
  options.signal.removeEventListener("abort", onAbort);
}
```

### Usage Tracking from Stream Events

```typescript
// Source: reference project message_start/message_delta handling
if (event.type === "message_start") {
  const usage = event.message?.usage;
  output.usage.input = usage?.input_tokens ?? 0;
  output.usage.output = usage?.output_tokens ?? 0;
  output.usage.cacheRead = usage?.cache_read_input_tokens ?? 0;
  output.usage.cacheWrite = usage?.cache_creation_input_tokens ?? 0;
  output.usage.totalTokens =
    output.usage.input +
    output.usage.output +
    output.usage.cacheRead +
    output.usage.cacheWrite;
  calculateCost(model, output.usage);
}

if (event.type === "message_delta") {
  output.stopReason = mapStopReason(event.delta?.stop_reason);
  const usage = event.usage ?? {};
  if (usage.input_tokens != null) output.usage.input = usage.input_tokens;
  if (usage.output_tokens != null) output.usage.output = usage.output_tokens;
  output.usage.totalTokens =
    output.usage.input +
    output.usage.output +
    output.usage.cacheRead +
    output.usage.cacheWrite;
  calculateCost(model, output.usage);
}
```

### Stop Reason Mapping

```typescript
// Source: reference project mapStopReason
function mapStopReason(
  reason: string | undefined,
): "stop" | "length" | "toolUse" {
  switch (reason) {
    case "tool_use":
      return "toolUse";
    case "max_tokens":
      return "length";
    case "end_turn":
    default:
      return "stop";
  }
}
```

### System Prompt: AGENTS.md Loading + Sanitization

```typescript
// Source: reference project extractAgentsAppend + sanitizeAgentsContent
function resolveAgentsMdPath(): string | undefined {
  // Walk up from cwd looking for AGENTS.md
  let current = resolve(process.cwd());
  while (true) {
    const candidate = join(current, "AGENTS.md");
    if (existsSync(candidate)) return candidate;
    const parent = dirname(current);
    if (parent === current) break;
    current = parent;
  }
  // Fall back to global path
  const globalPath = join(homedir(), ".pi", "agent", "AGENTS.md");
  if (existsSync(globalPath)) return globalPath;
  return undefined;
}

function sanitizeAgentsContent(content: string): string {
  let sanitized = content;
  sanitized = sanitized.replace(/~\/\.pi\b/gi, "~/.claude");
  sanitized = sanitized.replace(/(^|[\s'"`])\.pi\//g, "$1.claude/");
  sanitized = sanitized.replace(/\b\.pi\b/gi, ".claude");
  sanitized = sanitized.replace(/\bpi\b/gi, "environment");
  return sanitized;
}
```

### Done Event Structure

```typescript
// Source: reference project final event emission
stream.push({
  type: "done",
  reason:
    output.stopReason === "toolUse"
      ? "toolUse"
      : output.stopReason === "length"
        ? "length"
        : "stop",
  message: output,
});
stream.end();
```

## State of the Art

| Old Approach                                        | Current Approach                                                   | When Changed        | Impact                                                                                            |
| --------------------------------------------------- | ------------------------------------------------------------------ | ------------------- | ------------------------------------------------------------------------------------------------- |
| Claude Agent SDK transport (`query()`)              | Direct CLI subprocess (`claude -p` + stream-json)                  | This project (2026) | Full control over wire protocol, no SDK dependency, but must handle subprocess lifecycle manually |
| `--output-format json` (wait for complete response) | `--output-format stream-json --verbose --include-partial-messages` | 2025                | Real-time token streaming instead of waiting for complete response                                |
| `shell: true` for Windows subprocess                | `cross-spawn` package                                              | Established pattern | Avoids command injection risk from `shell: true`                                                  |
| Manual `data` event buffering for NDJSON            | `readline.createInterface()`                                       | Established pattern | Eliminates partial line bugs (Pitfall 2)                                                          |

**Known CLI limitations:**

- Extended thinking with `maxThinkingTokens` disables `StreamEvent` emission in the Agent SDK. Unclear if this applies to raw CLI stream-json mode. Needs experimental validation in Phase 3.
- The `--include-partial-messages` flag is required alongside `--output-format stream-json` for token-by-token streaming events.

## Open Questions

1. **Stdin lifecycle in stream-json mode**
   - What we know: Stream-json input mode keeps stdin open for multiple messages. The user message is one NDJSON line, control_responses are additional lines.
   - What's unclear: Does the CLI start processing immediately after receiving the user message, or does it wait for some signal? The reference project uses the SDK's `query()` which abstracts this away.
   - Recommendation: Send the user message and observe if streaming begins. If not, experiment with sending an empty line or a specific end-of-input signal. The `buildPromptStream` function in the reference project yields a single user message and the SDK handles the rest.

2. **Model ID compatibility with subscription auth**
   - What we know: The reference project passes model IDs from `getModels("anthropic")` directly to the SDK and it works. The CLI accepts `--model` with values like `claude-sonnet-4-6` or `sonnet`.
   - What's unclear: Whether all model IDs from `getModels("anthropic")` (which may include dated versions like `claude-sonnet-4-5-20250929`) work with the CLI's `--model` flag.
   - Recommendation: Test with several model IDs. If dated versions fail, strip the date suffix. Log the model ID for debugging.

3. **Stderr capture strategy**
   - What we know: stderr contains diagnostic output, not protocol data. Non-zero exit codes (combined with no result event) indicate errors.
   - What's unclear: How much stderr noise to expect and whether it's useful for error reporting to the user.
   - Recommendation: Capture stderr into a buffer. On subprocess error (no result event + non-zero exit), include stderr in the error message. Otherwise discard.

## Validation Architecture

### Test Framework

| Property           | Value                                                                                     |
| ------------------ | ----------------------------------------------------------------------------------------- |
| Framework          | Vitest (standard for TypeScript projects in Node.js 22+; no existing test infrastructure) |
| Config file        | none -- see Wave 0                                                                        |
| Quick run command  | `npx vitest run --reporter=verbose`                                                       |
| Full suite command | `npx vitest run`                                                                          |

### Phase Requirements -> Test Map

| Req ID  | Behavior                                                        | Test Type         | Automated Command                                                        | File Exists? |
| ------- | --------------------------------------------------------------- | ----------------- | ------------------------------------------------------------------------ | ------------ |
| PROV-01 | Provider registers with correct ID and streamSimple handler     | unit              | `npx vitest run tests/provider.test.ts -t "registers provider"`          | -- Wave 0    |
| PROV-02 | Models derived from getModels("anthropic") with correct fields  | unit              | `npx vitest run tests/provider.test.ts -t "exposes models"`              | -- Wave 0    |
| PROV-03 | streamSimple returns AssistantMessageEventStream                | unit              | `npx vitest run tests/provider.test.ts -t "streamSimple returns stream"` | -- Wave 0    |
| PROC-01 | Subprocess spawned with correct flags                           | unit (mock spawn) | `npx vitest run tests/process-manager.test.ts -t "spawn flags"`          | -- Wave 0    |
| STRM-01 | NDJSON parsing handles valid JSON, malformed lines, debug noise | unit              | `npx vitest run tests/stream-parser.test.ts -t "NDJSON parsing"`         | -- Wave 0    |
| STRM-02 | Text stream events bridged correctly (text_start/delta/end)     | unit              | `npx vitest run tests/event-bridge.test.ts -t "text events"`             | -- Wave 0    |
| HIST-01 | Flattened prompt built from conversation history                | unit              | `npx vitest run tests/prompt-builder.test.ts -t "builds prompt"`         | -- Wave 0    |

### Sampling Rate

- **Per task commit:** `npx vitest run --reporter=verbose`
- **Per wave merge:** `npx vitest run`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps

- [ ] `vitest.config.ts` -- Vitest configuration file
- [ ] `tests/provider.test.ts` -- PROV-01, PROV-02, PROV-03
- [ ] `tests/process-manager.test.ts` -- PROC-01
- [ ] `tests/stream-parser.test.ts` -- STRM-01
- [ ] `tests/event-bridge.test.ts` -- STRM-02
- [ ] `tests/prompt-builder.test.ts` -- HIST-01
- [ ] Framework install: `npm install --save-dev vitest`

## Sources

### Primary (HIGH confidence)

- [Claude Code CLI Reference](https://code.claude.com/docs/en/cli-reference) -- All CLI flags, auth commands, stream-json format
- [Agent SDK Streaming Output](https://platform.claude.com/docs/en/agent-sdk/streaming-output) -- StreamEvent types, message flow, content_block events
- [Agent SDK Permissions](https://platform.claude.com/docs/en/agent-sdk/permissions) -- Permission modes, canUseTool callback, control protocol
- [Agent SDK User Input](https://platform.claude.com/docs/en/agent-sdk/user-input) -- canUseTool response format (allow/deny with behavior field)
- [claude-agent-sdk-pi source code](https://github.com/prateekmedia/claude-agent-sdk-pi) -- Complete reference implementation (cloned and analyzed)
- [pi-mono custom-provider docs](https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/custom-provider.md) -- registerProvider API, model definitions, streamSimple contract
- [pi-mono extension types](https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/src/core/extensions/types.ts) -- ExtensionAPI, ProviderConfig types

### Secondary (MEDIUM confidence)

- [Inside the Claude Agent SDK (substack)](https://buildwithaws.substack.com/p/inside-the-claude-agent-sdk-from) -- Wire protocol reverse-engineering: control_request/control_response JSON formats
- [Claude Code stream-json event types issue #24596](https://github.com/anthropics/claude-code/issues/24596) -- Event type reference gap (closed as stale)

### Tertiary (LOW confidence)

- [Claude Code process hang issue #25629](https://github.com/anthropics/claude-code/issues/25629) -- Kill-after-result workaround
- [Claude Code debug stdout corruption issue #12007](https://github.com/anthropics/claude-code/issues/12007) -- Non-JSON lines on stdout
- [Claude SDK Windows stdin issue #208](https://github.com/anthropics/claude-agent-sdk-python/issues/208) -- Windows stdin flush behavior

## Metadata

**Confidence breakdown:**

- Standard stack: HIGH -- Node.js built-ins + cross-spawn are well-documented, established tools. Reference project validates the approach.
- Architecture: HIGH -- Reference project provides complete working pattern. This phase only replaces the SDK transport with direct subprocess, keeping all other architecture identical.
- Pitfalls: HIGH -- Multiple documented GitHub issues with reproduction steps. Kill-after-result is the established workaround.
- Wire protocol: MEDIUM -- Observed via reference project + reverse-engineering blog post. Official docs lack complete event type reference (issue #24596 closed as stale).

**Research date:** 2026-03-13
**Valid until:** 2026-04-13 (30 days -- CLI flags are stable; pi provider API is stable)
