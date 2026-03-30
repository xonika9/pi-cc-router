# Architecture Patterns

**Domain:** CLI subprocess LLM provider extension for pi coding agent
**Researched:** 2026-03-13

## Recommended Architecture

This extension acts as a **bridge** between pi's provider interface and the Claude Code CLI subprocess. It translates pi's `streamSimple` event contract into spawning a `claude -p` subprocess, feeding it a flattened conversation prompt via stream-json input, intercepting tool-use proposals via the control protocol, and streaming Claude API events back as pi `AssistantMessageEventStream` events.

### High-Level Architecture

```
+------------------+        +---------------------+        +-------------------+
|  pi coding agent |        |  pi-claude-cli      |        |  claude -p        |
|  (host runtime)  |        |  (extension)        |        |  (subprocess)     |
|                  |        |                     |        |                   |
|  registerProvider|------->| streamSimple()      |        |                   |
|                  |        |   |                 |        |                   |
|                  |        |   +-> PromptBuilder |        |                   |
|                  |        |   +-> ProcessMgr ---|------->| stdin (NDJSON)    |
|                  |        |   +-> StreamParser <|--------| stdout (NDJSON)   |
|                  |        |   +-> ToolRouter    |        |                   |
|  EventStream  <--|--------| EventBridge         |        |                   |
|                  |        |                     |        |                   |
|  tool_call event |------->| ToolMapper          |        |                   |
|  tool_result     |        |   (name+arg xlation)|        |                   |
|                  |        |                     |        |                   |
|  custom tools    |        | McpToolProxy -------|------->| MCP stdio server  |
|  (registered)    |        |   (exposes pi tools)|        | (--mcp-config)    |
+------------------+        +---------------------+        +-------------------+
```

### Component Boundaries

| Component                       | Responsibility                                                                                                                                                                                                                                                | Communicates With                                                                                        |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| **ExtensionEntry** (`index.ts`) | Extension lifecycle: registers provider, wires up models, handles activation/deactivation                                                                                                                                                                     | pi `ExtensionAPI`, all internal components                                                               |
| **PromptBuilder**               | Converts pi conversation history (messages, tool calls, tool results) into a flattened text prompt for `claude -p`                                                                                                                                            | ExtensionEntry (receives context), ProcessManager (provides prompt)                                      |
| **ProcessManager**              | Spawns `claude -p` subprocess, writes NDJSON to stdin, reads NDJSON from stdout, manages subprocess lifecycle                                                                                                                                                 | PromptBuilder (receives prompt), StreamParser (feeds raw lines), ToolRouter (sends control_responses)    |
| **StreamParser**                | Parses NDJSON lines from subprocess stdout, classifies messages as `stream_event` (API events) or `control_request` (tool approval)                                                                                                                           | ProcessManager (reads lines), EventBridge (forwards API events), ToolRouter (forwards control_requests)  |
| **ToolRouter**                  | Decides tool approval/denial based on tool name prefix: deny built-in tools (pi executes them), allow `mcp__` prefixed tools (Claude executes MCP calls)                                                                                                      | StreamParser (receives control_requests), ProcessManager (writes control_responses to stdin)             |
| **EventBridge**                 | Translates Claude API streaming events (`content_block_start/delta/stop`, `message_start/delta/stop`) into pi `AssistantMessageEventStream` events (`text_start/delta/end`, `toolcall_start/delta/end`, `thinking_start/delta/end`, `start`, `done`, `error`) | StreamParser (receives parsed events), pi EventStream (pushes translated events)                         |
| **ToolMapper**                  | Bidirectional translation of tool names (Claude `Read` <-> pi `read`, Claude `Glob` <-> pi `find`) and arguments (`file_path` <-> `path`, `old_string` <-> `oldText`)                                                                                         | EventBridge (translates outbound tool_call events), PromptBuilder (translates inbound tool_result names) |
| **McpToolProxy**                | Stdio MCP server that exposes pi's custom-registered tools to the Claude subprocess so Claude can call them via `mcp__custom-tools__<name>`                                                                                                                   | Claude subprocess (MCP client), pi tool registry (reads active custom tools)                             |

## Data Flow

### Primary Request Flow (per LLM turn)

```
1. pi calls streamSimple(model, context, options)
   |
2. PromptBuilder flattens context.messages into text prompt
   - USER: <user message>
   - ASSISTANT: <assistant response>
   - TOOL RESULT [tool_name]: <result content>
   |
3. ProcessManager spawns: claude -p "<prompt>"
     --input-format stream-json
     --output-format stream-json
     --verbose
     --include-partial-messages
     --model <model.id>
     --mcp-config <tempMcpConfig.json>   (if custom tools exist)
     [--strict-mcp-config false]          (if user wants .mcp.json to load)
   |
4. StreamParser reads stdout line-by-line (NDJSON)
   |
   +---> stream_event line --> EventBridge
   |     |
   |     +-> message_start      --> push { type: "start", partial }
   |     +-> content_block_start
   |     |   +-> type: "text"   --> push { type: "text_start" }
   |     |   +-> type: "tool_use" --> push { type: "toolcall_start" }
   |     |       (ToolMapper translates name+args)
   |     |   +-> type: "thinking" --> push { type: "thinking_start" }
   |     +-> content_block_delta
   |     |   +-> text_delta     --> push { type: "text_delta" }
   |     |   +-> input_json_delta --> push { type: "toolcall_delta" }
   |     |   +-> thinking_delta --> push { type: "thinking_delta" }
   |     +-> content_block_stop --> push { type: "text_end" / "toolcall_end" / "thinking_end" }
   |     +-> message_delta      --> update usage/stopReason
   |     +-> message_stop       --> push { type: "done", message }
   |
   +---> control_request line --> ToolRouter
         |
         +-> subtype: "can_use_tool"
         |   +-> tool_name starts with "mcp__"?
         |   |   YES --> write control_response { behavior: "allow" } to stdin
         |   |   NO  --> write control_response { behavior: "deny" } to stdin
         |   |           EventBridge pushes toolcall events with mapped name/args
         |
5. Subprocess exits when message completes
   |
6. EventBridge pushes final { type: "done", message } and calls stream.end()
```

### Control Protocol Messages (NDJSON on stdin/stdout)

**Subprocess stdout -- control_request:**

```json
{
  "type": "control_request",
  "request_id": "req_1_abc123",
  "request": {
    "subtype": "can_use_tool",
    "tool_name": "Bash",
    "input": { "command": "ls /home" }
  }
}
```

**Extension writes to stdin -- control_response (deny):**

```json
{
  "type": "control_response",
  "request_id": "req_1_abc123",
  "response": {
    "subtype": "success",
    "response": { "behavior": "deny" }
  }
}
```

**Extension writes to stdin -- control_response (allow MCP tool):**

```json
{
  "type": "control_response",
  "request_id": "req_1_abc123",
  "response": {
    "subtype": "success",
    "response": { "behavior": "allow" }
  }
}
```

### Tool Name Mapping (Bidirectional)

| Claude Built-in | pi Equivalent | Arg Translations                                                               |
| --------------- | ------------- | ------------------------------------------------------------------------------ |
| `Read`          | `read`        | `file_path` <-> `path`                                                         |
| `Write`         | `write`       | `file_path` <-> `path`, `content` <-> `content`                                |
| `Edit`          | `edit`        | `file_path` <-> `path`, `old_string` <-> `oldText`, `new_string` <-> `newText` |
| `Bash`          | `bash`        | `command` <-> `command`                                                        |
| `Grep`          | `grep`        | `pattern` <-> `pattern`, `path` <-> `path`                                     |
| `Glob`          | `find`        | `pattern` <-> `pattern`, `path` <-> `path`                                     |

Confidence: MEDIUM -- mapping details inferred from reference project README and pi extension docs. The exact argument field names should be verified against current pi and Claude CLI versions at implementation time.

### Custom Tool MCP Proxy Flow

```
1. On extension load:
   - Enumerate pi's registered custom tools via pi.getAllTools()
   - Filter out built-in tools (read, write, edit, bash, grep, find)
   - If custom tools exist, create an MCP tool proxy

2. McpToolProxy creates a stdio MCP server process
   - Listens on stdin, responds on stdout (MCP JSON-RPC protocol)
   - Implements tools/list handler: returns custom tool definitions
   - Implements tools/call handler: receives call, returns denied/error
     (Claude proposes the call, pi actually executes it --
      the proxy only needs to expose tool schemas so Claude knows
      the tools exist; execution is blocked via control_request denial
      and the tool_call event is forwarded to pi for execution)

3. ProcessManager writes a temp mcp-config.json:
   {
     "mcpServers": {
       "custom-tools": {
         "type": "stdio",
         "command": "node",
         "args": ["path/to/mcp-proxy.js"]
       }
     }
   }

4. claude -p is spawned with --mcp-config <temp-config>
   - Claude discovers mcp__custom-tools__<toolName> tools
   - When Claude proposes using one, control_request fires
   - ToolRouter allows mcp__ prefixed tools (Claude executes them via MCP)

   ALTERNATIVE APPROACH (simpler):
   - The MCP proxy returns tool results directly from pi's tool execution
   - Instead of allowing the control_request AND having pi execute separately,
     the proxy intercepts the MCP call, delegates to pi's tool executor,
     and returns the result through MCP -- making it transparent to Claude
```

**Confidence note on MCP proxy:** The reference project uses the SDK's `createSdkMcpServer()` which handles this transparently. Without the SDK, we need to implement an equivalent. Two viable approaches exist (detailed above). The simpler approach -- where the MCP proxy server calls pi's tool executor and returns results via MCP -- avoids complex coordination between MCP allow/deny and pi's separate tool execution. This is a key architectural decision to resolve in Phase 1.

## Component Details

### ExtensionEntry (`index.ts`)

The entry point exports a default function receiving `ExtensionAPI`. Responsibilities:

```typescript
export default function (pi: ExtensionAPI) {
  // 1. Get available Claude models
  const models = getModels("anthropic"); // or hardcoded list

  // 2. Register as custom provider
  pi.registerProvider("claude-cli", {
    streamSimple: (model, context, options) => {
      return streamViaCli(model, context, options, pi);
    },
    models: [
      {
        id: "claude-opus-4-6",
        name: "Claude Opus 4.6",
        reasoning: true,
        input: ["text", "image"],
        cost: { input: 15, output: 75, cacheRead: 1.5, cacheWrite: 18.75 },
        contextWindow: 200000,
        maxTokens: 32000,
      },
      // ... other models
    ],
  });
}
```

### PromptBuilder

Converts pi's conversation context into a single prompt string. The stateless model means every request replays the full conversation.

Key considerations:

- Messages are role-tagged: `USER:`, `ASSISTANT:`, `TOOL RESULT [name]:`
- Tool call proposals from the assistant are included so Claude sees its own prior reasoning
- System prompt from pi context may be appended via `--append-system-prompt` or embedded in the prompt
- Image content blocks need special handling (Claude CLI supports image inputs but the encoding/format needs validation)

### ProcessManager

Manages the `child_process.spawn()` lifecycle:

```typescript
// Spawn args construction
const args = [
  "-p",
  prompt,
  "--input-format",
  "stream-json",
  "--output-format",
  "stream-json",
  "--verbose",
  "--include-partial-messages",
  "--model",
  model.id,
];

if (mcpConfigPath) {
  args.push("--mcp-config", mcpConfigPath);
}

if (!strictMcpConfig) {
  // Don't add --strict-mcp-config, let .mcp.json load
} else {
  args.push("--strict-mcp-config");
}

const proc = spawn("claude", args, {
  stdio: ["pipe", "pipe", "pipe"],
  // Windows: shell: true may be needed
});
```

Platform considerations:

- **Windows:** May need `shell: true` or explicit `.cmd` extension for the `claude` command
- **AbortSignal:** Must handle `options?.signal` from pi to kill the subprocess on cancellation
- **stderr:** Capture for error reporting but do not parse as protocol data

### StreamParser

Reads stdout line-by-line, parsing each as JSON:

```typescript
// Pseudocode
for await (const line of readLines(proc.stdout)) {
  const msg = JSON.parse(line);

  if (msg.type === "stream_event") {
    eventBridge.handleStreamEvent(msg.event);
  } else if (msg.type === "control_request") {
    toolRouter.handleControlRequest(msg, proc.stdin);
  }
  // Ignore other message types (system, result, etc.)
}
```

The `--verbose` and `--include-partial-messages` flags are critical -- without them, stream_event messages with granular content_block_delta events are not emitted.

### EventBridge

The core translation layer. Maintains state for:

- Current `AssistantMessage` being built (the `partial` object)
- Content block index tracking (maps Claude's content indices to pi's)
- Usage accumulation (input/output/cache tokens)

State machine per content block:

```
content_block_start(type=text)      -> push text_start
content_block_delta(text_delta)     -> push text_delta (repeatable)
content_block_stop                  -> push text_end

content_block_start(type=tool_use)  -> push toolcall_start (with mapped name)
content_block_delta(input_json_delta) -> push toolcall_delta (accumulate JSON)
content_block_stop                  -> push toolcall_end (with parsed args, mapped)

content_block_start(type=thinking)  -> push thinking_start
content_block_delta(thinking_delta) -> push thinking_delta
content_block_stop                  -> push thinking_end
```

### ToolRouter

Simple decision logic, but critical for correctness:

```typescript
function shouldAllowTool(toolName: string): boolean {
  // MCP tools are executed by Claude via the MCP server -- allow them
  if (toolName.startsWith("mcp__")) return true;

  // Built-in tools are denied -- pi executes them natively
  // The denied tool_use is surfaced as a toolcall event to pi
  return false;
}
```

When a tool is denied:

1. Write `control_response` with `behavior: "deny"` to subprocess stdin
2. The EventBridge has already captured the tool_use content block from the stream
3. Pi receives the `toolcall_start/delta/end` events and executes the tool itself
4. The tool result gets included in the NEXT `streamSimple` call's context (since sessions are stateless)

### ToolMapper

Stateless bidirectional mapping. Two functions:

```typescript
// Claude -> pi (outbound: when Claude proposes a tool call)
function mapToolCallToPi(
  claudeName: string,
  claudeArgs: Record<string, any>,
): { name: string; args: Record<string, any> };

// pi -> Claude (inbound: when building prompt with prior tool results)
function mapToolResultToClaude(
  piName: string,
  piArgs: Record<string, any>,
): { name: string; args: Record<string, any> };
```

### McpToolProxy

The most architecturally significant new component (not in the SDK path). Two implementation approaches:

**Approach A: Schema-only proxy (simpler, recommended first)**

- MCP server exposes tool schemas from `pi.getAllTools()` minus built-ins
- `tools/call` handler returns an error/empty result (tool is never actually called via MCP)
- Claude knows the tools exist (proposes them), but execution is denied via control_request
- Pi executes the tool natively when it receives the toolcall event
- Problem: Claude may get confused by MCP tool call failures if it expects a result

**Approach B: Delegating proxy (more complex, likely needed)**

- MCP server exposes tool schemas from `pi.getAllTools()`
- `tools/call` handler delegates to pi's tool execution (via extension API or direct invocation)
- Returns the real tool result through MCP to Claude
- Control_request for `mcp__` tools is allowed (Claude calls them via MCP)
- Pi does NOT separately execute these tools (avoids double execution)
- Requires a way to invoke pi tool execution from within the MCP server process

**Recommendation:** Start with Approach A for simplicity. If Claude's behavior is degraded by denied MCP tool calls, switch to Approach B.

## Patterns to Follow

### Pattern 1: Stateless Subprocess per Request

**What:** Spawn a fresh `claude -p` process for each `streamSimple` call with full conversation history replayed
**When:** Every LLM turn
**Why:** Matches reference project pattern. Avoids session state complexity. Each call is independent.

```typescript
// Each streamSimple call is a fresh subprocess
function streamViaCli(model, context, options, pi) {
  const stream = createAssistantMessageEventStream();

  (async () => {
    const prompt = buildPrompt(context);
    const proc = spawnClaude(model, prompt, options);

    // Parse and bridge events...

    proc.on("exit", () => stream.end());
  })();

  return stream;
}
```

### Pattern 2: NDJSON Line-Delimited Parsing

**What:** Read subprocess stdout as newline-delimited JSON, one complete object per line
**When:** All subprocess communication
**Why:** The stream-json protocol guarantees one JSON object per line.

```typescript
import { createInterface } from "node:readline";

const rl = createInterface({ input: proc.stdout, crlfDelay: Infinity });
for await (const line of rl) {
  if (line.trim()) {
    const msg = JSON.parse(line);
    // handle msg...
  }
}
```

### Pattern 3: Event Stream State Machine

**What:** Track content block state to correctly sequence pi events
**When:** Translating Claude API events to pi events
**Why:** Claude API events are indexed by content block; pi expects sequential start/delta/end groups.

### Pattern 4: Graceful Subprocess Cleanup

**What:** Kill subprocess on abort signal, handle unexpected exits
**When:** Cancellation, errors, timeouts
**Why:** Orphaned processes leak resources.

```typescript
options?.signal?.addEventListener("abort", () => {
  proc.kill("SIGTERM");
});

proc.on("error", (err) => {
  stream.push({ type: "error", reason: "error", error: output });
  stream.end();
});
```

## Anti-Patterns to Avoid

### Anti-Pattern 1: Persistent Subprocess Sessions

**What:** Keeping a claude subprocess alive across multiple `streamSimple` calls
**Why bad:** The stream-json input mode supports persistent sessions, but this adds significant complexity: session state management, reconnection on crash, prompt deduplication, and the reference project doesn't use it. Risk of subtle state corruption bugs.
**Instead:** Stateless subprocess per request. Accept the token cost of replaying history.

### Anti-Pattern 2: Parsing stderr as Protocol Data

**What:** Treating subprocess stderr output as NDJSON protocol messages
**Why bad:** stderr contains diagnostic output, warnings, and error messages -- not structured protocol data
**Instead:** Log stderr for debugging, only parse stdout as NDJSON protocol

### Anti-Pattern 3: Synchronous Subprocess Communication

**What:** Using `execSync` or blocking reads for subprocess I/O
**Why bad:** Blocks the Node.js event loop, prevents streaming, breaks pi's real-time token display
**Instead:** Async spawn with piped stdio, readline interface for line-by-line processing

### Anti-Pattern 4: Hardcoded Tool Mappings Without Validation

**What:** Assuming tool name/argument mappings are static and never change
**Why bad:** Both pi and Claude CLI evolve independently; tool names or argument schemas may change
**Instead:** Define mappings as configuration objects, log warnings on unknown tools, fail gracefully

### Anti-Pattern 5: Double Tool Execution

**What:** Both allowing an MCP tool call AND having pi execute the same tool
**Why bad:** Tool runs twice, potentially with side effects (e.g., writing a file twice)
**Instead:** Either the MCP proxy handles execution (allow in control_request) OR pi handles it (deny in control_request), never both

## Scalability Considerations

| Concern             | At MVP                                    | At Production                           | Notes                                                         |
| ------------------- | ----------------------------------------- | --------------------------------------- | ------------------------------------------------------------- |
| Subprocess overhead | 1 process per turn, ~50KB tokens replayed | Same (stateless is the model)           | Token cost is the subscription cost, not API billing          |
| Process cleanup     | Kill on abort                             | Kill on abort + timeout watchdog        | Prevent orphaned processes on crashes                         |
| MCP proxy lifecycle | Start/stop per subprocess                 | Cache proxy process, reuse across calls | MCP server startup has latency                                |
| Tool mapping        | 6 hardcoded mappings                      | Configurable mapping table              | Pi may add/rename tools                                       |
| Platform compat     | Test on one OS                            | Windows + macOS + Linux testing         | `spawn` behavior differs, especially `shell: true` on Windows |
| Error recovery      | Log and surface error event               | Retry logic, fallback to error state    | Subprocess may crash mid-stream                               |

## Suggested Build Order (Dependencies)

Build order is driven by component dependencies. Each phase depends on the previous.

### Phase 1: Core Subprocess Bridge (foundational)

**Components:** ExtensionEntry, ProcessManager, StreamParser (text-only), EventBridge (text events only)
**Delivers:** Extension registers as provider, spawns subprocess, streams text responses back to pi
**Why first:** Everything else builds on the ability to spawn and communicate with the subprocess
**Validates:** Stream-json protocol works, pi provider registration works, basic streaming works

### Phase 2: Tool Denial and Mapping

**Components:** ToolRouter, ToolMapper, EventBridge (add toolcall events)
**Delivers:** Claude proposes tools, extension denies built-in tools, pi receives mapped tool_call events and executes them
**Why second:** Tool handling is the core value proposition -- without it, Claude has no tools
**Validates:** Control protocol works, tool name/arg mapping is correct, pi executes tools from Claude's proposals

### Phase 3: Extended Thinking and Usage

**Components:** EventBridge (add thinking events), usage tracking, cost calculation
**Delivers:** Thinking tokens stream to pi, usage/cost metrics are accurate
**Why third:** Thinking support requires the streaming infrastructure from Phase 1-2 but is independent of tool handling
**Validates:** Thinking content blocks parse correctly, usage accumulation works

### Phase 4: Custom Tool MCP Proxy

**Components:** McpToolProxy
**Delivers:** Pi's custom-registered tools are visible to Claude and can be called
**Why fourth:** This is the most complex new component. Requires the control protocol (Phase 2) to work correctly first
**Validates:** MCP server spawns correctly, Claude discovers custom tools, tool calls route correctly

### Phase 5: Platform Hardening

**Components:** Cross-platform subprocess handling, error recovery, edge cases
**Delivers:** Works on Windows/macOS/Linux, handles crashes gracefully, proper cleanup
**Why last:** Correctness and robustness polish after core functionality is proven

```
Phase 1: ExtensionEntry + ProcessManager + StreamParser + EventBridge(text)
    |
    v
Phase 2: ToolRouter + ToolMapper + EventBridge(toolcall)
    |
    v
Phase 3: EventBridge(thinking) + Usage tracking
    |
    v
Phase 4: McpToolProxy
    |
    v
Phase 5: Platform hardening + error recovery
```

## Key Architectural Decisions to Resolve

| Decision               | Options                                              | Recommendation                                                                    | Confidence                                                       |
| ---------------------- | ---------------------------------------------------- | --------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| Prompt format          | Flattened text vs structured messages                | Flattened text (matches reference project)                                        | HIGH -- reference project validates this                         |
| MCP proxy approach     | Schema-only vs delegating proxy                      | Start with schema-only (Approach A), upgrade if needed                            | MEDIUM -- depends on Claude's behavior when MCP tools are denied |
| System prompt handling | `--append-system-prompt` flag vs embedded in prompt  | `--append-system-prompt` flag (cleaner separation)                                | MEDIUM -- needs testing                                          |
| Windows subprocess     | `spawn("claude", ...)` vs `spawn("claude.cmd", ...)` | Detect platform, use `.cmd` on Windows or `shell: true`                           | HIGH -- standard Node.js pattern                                 |
| MCP config persistence | Temp file per request vs persistent config           | Temp file per request (stateless model, clean up after)                           | HIGH -- matches stateless subprocess pattern                     |
| Image content handling | Base64 in prompt vs file reference                   | Needs investigation -- unclear if `claude -p` accepts images in stream-json input | LOW -- requires experimentation                                  |

## Sources

- [pi custom-provider docs](https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/custom-provider.md) -- HIGH confidence, official documentation
- [pi extensions docs](https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/extensions.md) -- HIGH confidence, official documentation
- [claude-agent-sdk-pi reference project](https://github.com/prateekmedia/claude-agent-sdk-pi) -- HIGH confidence, proven implementation
- [Claude Code CLI reference](https://code.claude.com/docs/en/cli-reference) -- HIGH confidence, official documentation
- [Claude Agent SDK streaming output](https://platform.claude.com/docs/en/agent-sdk/streaming-output) -- HIGH confidence, official documentation
- [Claude Agent SDK custom tools](https://platform.claude.com/docs/en/agent-sdk/custom-tools) -- HIGH confidence, official documentation
- [Inside the Claude Agent SDK (substack)](https://buildwithaws.substack.com/p/inside-the-claude-agent-sdk-from) -- MEDIUM confidence, reverse-engineered protocol details
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk) -- HIGH confidence, official MCP implementation
