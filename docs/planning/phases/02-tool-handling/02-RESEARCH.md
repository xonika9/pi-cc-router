# Phase 2: Tool Handling - Research

**Researched:** 2026-03-13
**Domain:** Claude CLI stream-json control protocol, tool name/argument mapping, tool call streaming events
**Confidence:** HIGH

## Summary

Phase 2 extends the Phase 1 stream parser and event bridge to handle Claude's tool_use content blocks and control protocol requests. The core pattern is "Claude proposes, pi executes": when Claude wants to use a tool, the CLI sends a `control_request` on stdout; our extension responds on stdin with `control_response` (deny for built-in tools, allow for MCP-prefixed tools). Tool names and arguments are mapped bidirectionally between Claude's format and pi's format using a centralized mapping table.

The reference project (`claude-agent-sdk-pi`) provides verified patterns for all mapping tables, JSON argument accumulation, and event emission. The key difference is that the reference project uses the Agent SDK's `canUseTool` callback, while we use the raw CLI control protocol over stdin/stdout. The wire format for control messages has been verified from the SDK documentation and architectural analysis: control_request includes a `request_id` for multiplexing.

**Primary recommendation:** Create a `tool-mapping.ts` module with a single-source-of-truth mapping table, a `control-handler.ts` module for control_request/control_response logic, and extend the existing `event-bridge.ts` to handle tool_use content blocks alongside existing text/thinking blocks.

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions

- Deny ALL non-MCP tools via `control_response` with `behavior: "deny"`
- Allow `mcp__` prefixed tools via `control_response` with `behavior: "allow"` -- these are Claude's own MCP server tools that pi cannot execute
- Deny message: `"Tool execution is unavailable in this environment."` (matches reference project)
- Deny each `control_request` individually as it arrives -- no batching, immediate response per request
- Allowed MCP tool calls still emit toolcall events to pi (pi can display them in the UI even though it did not execute them)
- Unknown/unmapped tools pass through unchanged -- if Claude proposes a tool not in the 6-tool mapping table, emit toolcall events with the original Claude tool name and args unmodified (matches reference project: no warning, no drop, no error)
- Bidirectional mapping for 6 known tools: Read<->read, Write<->write, Edit<->edit, Bash<->bash, Grep<->grep, Glob<->find
- Single source of truth mapping table -- derive both directions from one definition
- Handle the glob/find asymmetry: Claude's Glob maps to pi's `find`; pi's `find` AND `glob` both map back to Claude's `Glob`
- Only translate known renamed arguments + pass everything else through unchanged
- Known renames: `file_path`<->`path` (Read/Write/Edit), `old_string`<->`oldText` and `new_string`<->`newText` (Edit), `head_limit`<->`limit` (Grep)
- `toolcall_start` fires immediately on `content_block_start` (as soon as tool name is known)
- `toolcall_delta` streams raw JSON argument fragments as they arrive from `content_block_delta` events
- `toolcall_end` emits after `content_block_stop` with fully accumulated and parsed args object (with name/arg mapping applied)
- If accumulated JSON args fail to parse: emit `toolcall_end` with the raw unparsed string instead of a parsed object

### Claude's Discretion

- Internal buffering strategy for accumulating JSON argument fragments
- Exact control_response JSON structure (beyond the decided behavior/message fields)
- Error handling for malformed control_request messages
- Timing/ordering of toolcall events relative to text events in multi-content-block responses

### Deferred Ideas (OUT OF SCOPE)

None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>

## Phase Requirements

| ID      | Description                                                                                                                           | Research Support                                                                                                                    |
| ------- | ------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| STRM-03 | Send `control_response` on stdin to deny/allow tool execution in response to `control_request` on stdout                              | Wire protocol format verified (request_id + response object); deny/allow behavior patterns from reference project and SDK docs      |
| TOOL-01 | Deny all built-in tool execution requests via control protocol (`behavior: "deny"`)                                                   | Exact deny message constant, PermissionResult type from SDK docs, reference project `canUseTool` pattern                            |
| TOOL-02 | Allow `mcp__` prefixed tool execution via control protocol (`behavior: "allow"`)                                                      | MCP prefix detection logic from reference project (`mapToolName`), allow response format from SDK PermissionResult                  |
| TOOL-03 | Map tool names bidirectionally between Claude and pi (Read<->read, Write<->write, Edit<->edit, Bash<->bash, Grep<->grep, Glob<->find) | Complete mapping tables extracted from reference project (SDK_TO_PI_TOOL_NAME/PI_TO_SDK_TOOL_NAME), single-source approach designed |
| TOOL-04 | Translate tool arguments between Claude and pi formats (file_path<->path, old_string<->oldText, head_limit<->limit)                   | Complete argument mapping per tool extracted from reference project's `mapToolArgs` function                                        |

</phase_requirements>

## Standard Stack

### Core

| Library          | Version  | Purpose                              | Why Standard                  |
| ---------------- | -------- | ------------------------------------ | ----------------------------- |
| Node.js readline | built-in | NDJSON line parsing (already in use) | Phase 1 established pattern   |
| vitest           | ^3.0.0   | Unit testing                         | Already configured in project |

### Supporting

| Library             | Version | Purpose                                    | When to Use                                             |
| ------------------- | ------- | ------------------------------------------ | ------------------------------------------------------- |
| @mariozechner/pi-ai | ^0.52.0 | ToolCall type, AssistantMessageEventStream | Peer dep, provides type definitions for toolcall events |

### Alternatives Considered

| Instead of                         | Could Use                     | Tradeoff                                                                                                                    |
| ---------------------------------- | ----------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Manual JSON.parse for partial JSON | streaming-json-parser library | Unnecessary complexity -- reference project just tries JSON.parse with fallback, no partial parsing needed during streaming |
| Separate mapping files per tool    | Single mapping table          | Single table is simpler, matches decision, easier to maintain                                                               |

**Installation:**
No new dependencies needed. All functionality uses built-in Node.js APIs and existing project dependencies.

## Architecture Patterns

### Recommended Project Structure

```
src/
├── tool-mapping.ts      # NEW: Single-source mapping table + translation functions
├── control-handler.ts   # NEW: control_request/control_response protocol handler
├── event-bridge.ts      # MODIFIED: Add tool_use content block handling
├── types.ts             # MODIFIED: Update ClaudeControlRequest, add ControlResponse type
├── provider.ts          # MODIFIED: Wire control handler into stream processing
├── stream-parser.ts     # UNCHANGED
├── process-manager.ts   # UNCHANGED
└── prompt-builder.ts    # MODIFIED: Add pi->Claude tool name/arg mapping for history
```

### Pattern 1: Single-Source Tool Mapping Table

**What:** Define one mapping table that derives both Claude-to-pi and pi-to-Claude directions.
**When to use:** For all tool name and argument translations.
**Example:**

```typescript
// Source: Reference project pattern, improved to single-source

interface ToolMapping {
  claude: string; // Claude's tool name (PascalCase)
  pi: string; // Pi's tool name (lowercase)
  args: Record<string, string>; // Claude arg name -> pi arg name (only renamed args)
}

const TOOL_MAPPINGS: ToolMapping[] = [
  { claude: "Read", pi: "read", args: { file_path: "path" } },
  { claude: "Write", pi: "write", args: { file_path: "path" } },
  {
    claude: "Edit",
    pi: "edit",
    args: { file_path: "path", old_string: "oldText", new_string: "newText" },
  },
  { claude: "Bash", pi: "bash", args: {} },
  { claude: "Grep", pi: "grep", args: { head_limit: "limit" } },
  { claude: "Glob", pi: "find", args: {} },
];

// Derive lookup tables from single source
const CLAUDE_TO_PI_NAME: Record<string, string> = {}; // lowercase claude -> pi
const PI_TO_CLAUDE_NAME: Record<string, string> = {}; // pi -> PascalCase claude
const CLAUDE_TO_PI_ARGS: Record<string, Record<string, string>> = {}; // per-tool arg renames
const PI_TO_CLAUDE_ARGS: Record<string, Record<string, string>> = {}; // reverse arg renames

for (const m of TOOL_MAPPINGS) {
  CLAUDE_TO_PI_NAME[m.claude.toLowerCase()] = m.pi;
  PI_TO_CLAUDE_NAME[m.pi] = m.claude;
  // Build arg maps
  CLAUDE_TO_PI_ARGS[m.claude.toLowerCase()] = m.args;
  const reverseArgs: Record<string, string> = {};
  for (const [from, to] of Object.entries(m.args)) {
    reverseArgs[to] = from;
  }
  PI_TO_CLAUDE_ARGS[m.pi] = reverseArgs;
}

// Handle glob/find asymmetry
PI_TO_CLAUDE_NAME["glob"] = "Glob";
```

### Pattern 2: Control Protocol Handler

**What:** Process `control_request` NDJSON messages, determine deny/allow, write `control_response` to stdin.
**When to use:** Every time a `control_request` message arrives on stdout.
**Example:**

```typescript
// Source: Wire protocol verified via SDK docs + architectural article

import type { ChildProcess } from "node:child_process";

const TOOL_EXECUTION_DENIED_MESSAGE =
  "Tool execution is unavailable in this environment.";
const MCP_PREFIX = "mcp__";

interface ControlRequest {
  type: "control_request";
  request_id: string;
  request: {
    subtype: "can_use_tool";
    tool_name: string;
    input: Record<string, unknown>;
  };
}

interface ControlResponse {
  type: "control_response";
  request_id: string;
  response: {
    subtype: "success";
    response: {
      behavior: "allow" | "deny";
      message?: string;
      updatedInput?: Record<string, unknown>;
    };
  };
}

function handleControlRequest(msg: ControlRequest, proc: ChildProcess): void {
  const toolName = msg.request.tool_name;
  const isMcp = toolName.startsWith(MCP_PREFIX);

  const response: ControlResponse = {
    type: "control_response",
    request_id: msg.request_id,
    response: {
      subtype: "success",
      response: isMcp
        ? { behavior: "allow" }
        : { behavior: "deny", message: TOOL_EXECUTION_DENIED_MESSAGE },
    },
  };

  proc.stdin!.write(JSON.stringify(response) + "\n");
}
```

### Pattern 3: Tool Use Content Block Streaming

**What:** Handle `tool_use` type in content_block_start/delta/stop to emit toolcall events.
**When to use:** When Claude streams a tool call alongside text/thinking content.
**Example:**

```typescript
// Source: Reference project event bridge (lines 700-796)

// On content_block_start with type "tool_use":
//   - Create ToolCall in output.content with id, mapped name, empty args, partialJson buffer
//   - Push toolcall_start event

// On content_block_delta with type "input_json_delta":
//   - Accumulate delta.partial_json into partialJson buffer
//   - Try JSON.parse on accumulated buffer (fallback to previous args)
//   - Push toolcall_delta with the raw delta string

// On content_block_stop:
//   - Final JSON.parse of accumulated partialJson
//   - Apply argument mapping (mapToolArgs)
//   - Push toolcall_end with complete ToolCall object
```

### Anti-Patterns to Avoid

- **Inline tool-specific switch/case for mappings:** Use the centralized mapping table instead. Scattered tool-specific logic is hard to maintain and error-prone when either Claude or pi adds/renames tools.
- **Batching control_response messages:** The decision requires immediate individual response per `control_request`. Batching could cause the CLI to stall waiting for responses.
- **Parsing partial JSON with a streaming parser:** The reference project simply tries `JSON.parse()` on the accumulated buffer and falls back to the previous value. This is simpler and sufficient.
- **Dropping unknown tool calls:** Unknown tools must pass through unchanged per the decision. Never silently drop or warn about tools not in the mapping table.

## Don't Hand-Roll

| Problem                 | Don't Build                      | Use Instead                                       | Why                                                                                                                          |
| ----------------------- | -------------------------------- | ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| Partial JSON parsing    | Custom streaming JSON parser     | `JSON.parse()` with try/catch fallback            | Reference project validates this approach; partial values are only for UI preview, final parse happens at content_block_stop |
| Tool name normalization | Complex regex/string matching    | Simple lowercase lookup + `startsWith("mcp__")`   | Only 6 tools + MCP prefix, no need for complex matching                                                                      |
| Argument mapping        | Generic schema-based transformer | Per-tool rename map with passthrough for unknowns | The mapping is small and static; a schema system would be over-engineering                                                   |

**Key insight:** The mapping tables are small (6 tools, ~6 argument renames) and change infrequently. A simple data-driven approach with derived lookup tables is more maintainable than any framework.

## Common Pitfalls

### Pitfall 1: Wrong Control Protocol Wire Format

**What goes wrong:** Sending a `control_response` with the wrong JSON structure causes the CLI to hang or error.
**Why it happens:** The wire format is not well-documented in public CLI docs. The SDK abstracts it with the `canUseTool` callback, so most developers never see the raw format.
**How to avoid:** Use the verified wire format: `{ type: "control_response", request_id: "...", response: { subtype: "success", response: { behavior: "deny", message: "..." } } }`. The `request_id` MUST match the incoming request's `request_id`.
**Warning signs:** CLI subprocess hangs after a tool call; no `content_block_stop` received after sending response.

### Pitfall 2: Missing request_id in Types

**What goes wrong:** The current `ClaudeControlRequest` type in `types.ts` does NOT include `request_id` or the nested `request` wrapper. Responding without a matching `request_id` will cause protocol desync.
**Why it happens:** The initial type definition was created based on partial documentation before the wire format was fully verified.
**How to avoid:** Update `ClaudeControlRequest` to include `request_id` and the `request` subobject with `subtype`, `tool_name`, and `input`. This is a breaking type change that must be done carefully.
**Warning signs:** TypeScript type errors when trying to read `msg.request_id` or `msg.request.tool_name`.

### Pitfall 3: Content Block Index Tracking Confusion

**What goes wrong:** Tool_use content blocks can appear at any index in the content array, interleaved with text blocks. Using the wrong index causes events to reference incorrect content.
**Why it happens:** Claude may output text before tool calls (e.g., "Let me read that file" then tool_use), so tool_use blocks are not always at index 0.
**How to avoid:** Track content blocks using Claude's `event.index` field (as the existing code already does for text blocks). The `contentIndex` in pi events should reference the position in `output.content`, not Claude's raw index.
**Warning signs:** `toolcall_end` event's `contentIndex` doesn't match the position of the ToolCall in `output.content`.

### Pitfall 4: Glob/Find Asymmetry

**What goes wrong:** Pi sends `find` as a tool name; the prompt builder maps it to `Glob`. But pi also has a `glob` tool name that also maps to `Glob`. If the reverse mapping only handles `find` -> `Glob`, `glob` -> `Glob` will be missed.
**Why it happens:** Pi has two tool names (`find` and `glob`) that both correspond to Claude's single `Glob` tool. The reference project handles this explicitly in `PI_TO_SDK_TOOL_NAME`.
**How to avoid:** In the pi-to-Claude direction, map both `find` and `glob` to `Glob`. The single-source table handles `find` naturally (it's the primary mapping); add `glob` as an alias.
**Warning signs:** History replay fails when pi sends a `glob` tool result that doesn't get mapped to `Glob` in the prompt.

### Pitfall 5: Argument Translation Must Be Selective

**What goes wrong:** Translating ALL arguments drops unknown/extra arguments that Claude or pi may send.
**Why it happens:** The reference project's `mapToolArgs` explicitly picks known fields per tool, dropping everything else. This is intentional for the SDK but NOT what we want.
**How to avoid:** Translate only the known renamed arguments, pass everything else through unchanged. Start with the input object, then rename the specific keys. Do NOT build a new object with only known fields.
**Warning signs:** Arguments like `offset`, `limit` (on Read), `glob` (on Grep), `timeout` (on Bash) disappear after translation.

### Pitfall 6: Prompt Builder Needs Reverse Mapping for Tool Results

**What goes wrong:** When pi replays conversation history containing tool results, tool names and arguments are in pi's format. Claude won't recognize `read` -- it expects `Read`. Arguments like `path` need to be `file_path`.
**Why it happens:** The prompt builder currently serializes tool calls as historical text (line 108-110 of prompt-builder.ts). When Phase 2 adds full tool handling, the prompt must use Claude's naming.
**How to avoid:** Apply PI_TO_CLAUDE_NAME and PI_TO_CLAUDE_ARGS mapping in the prompt builder when serializing tool call blocks from history.
**Warning signs:** Claude doesn't recognize tool results from previous turns; asks to "try again" or re-executes tools.

## Code Examples

Verified patterns from the reference project and official sources:

### Tool Name Mapping (Claude to Pi)

```typescript
// Source: Reference project lines 12-29, adapted to single-source

function mapClaudeToolNameToPi(claudeName: string): string {
  const normalized = claudeName.toLowerCase();
  return CLAUDE_TO_PI_NAME[normalized] ?? claudeName;
}

function mapPiToolNameToClaude(piName: string): string {
  return PI_TO_CLAUDE_NAME[piName] ?? piName;
}
```

### Argument Translation (Claude to Pi)

```typescript
// Source: Reference project mapToolArgs (lines 374-421), adapted for passthrough

function translateClaudeArgsToPi(
  toolName: string,
  args: Record<string, unknown>,
): Record<string, unknown> {
  const normalized = toolName.toLowerCase();
  const renames = CLAUDE_TO_PI_ARGS[normalized];
  if (!renames || Object.keys(renames).length === 0) return args;

  const result: Record<string, unknown> = {};
  for (const [key, value] of Object.entries(args)) {
    const newKey = renames[key] ?? key; // Rename if mapped, keep otherwise
    result[newKey] = value;
  }
  return result;
}
```

### Partial JSON Accumulation

```typescript
// Source: Reference project parsePartialJson (lines 542-549)

function parsePartialJson(
  accumulated: string,
  fallback: Record<string, unknown>,
): Record<string, unknown> {
  if (!accumulated) return fallback;
  try {
    return JSON.parse(accumulated);
  } catch {
    return fallback;
  }
}
```

### Tool Use in Content Block Start

```typescript
// Source: Reference project lines 700-711

// Inside handleContentBlockStart:
if (blockType === "tool_use") {
  const block = {
    type: "toolCall" as const,
    id: event.content_block!.id!,
    name: mapClaudeToolNameToPi(event.content_block!.name!),
    arguments: {} as Record<string, unknown>,
    partialJson: "",
    index: event.index ?? 0,
  };
  blocks.push(block);
  output.content.push({
    type: "toolCall" as const,
    id: block.id,
    name: block.name,
    arguments: block.arguments,
  });

  stream.push({
    type: "toolcall_start",
    contentIndex: output.content.length - 1,
    partial: output,
  });
}
```

### Tool Use in Content Block Delta (input_json_delta)

```typescript
// Source: Reference project lines 741-753

// Inside handleContentBlockDelta:
if (deltaType === "input_json_delta" && event.delta!.partial_json != null) {
  const idx = blocks.findIndex((b) => b.index === event.index);
  if (idx === -1) return;

  const block = blocks[idx];
  if (block.type === "toolCall") {
    block.partialJson += event.delta!.partial_json;
    block.arguments = parsePartialJson(block.partialJson, block.arguments);

    stream.push({
      type: "toolcall_delta",
      contentIndex: idx,
      delta: event.delta!.partial_json,
      partial: output,
    });
  }
}
```

### Tool Use in Content Block Stop

```typescript
// Source: Reference project lines 783-796

// Inside handleContentBlockStop:
if (block.type === "toolCall") {
  // Final parse and argument mapping
  const finalArgs = parsePartialJson(block.partialJson, block.arguments);
  block.arguments = translateClaudeArgsToPi(block.name, finalArgs);
  delete (block as any).partialJson;
  delete (block as any).index;

  const contentBlock = output.content[idx] as ToolCall;
  contentBlock.arguments = block.arguments;

  stream.push({
    type: "toolcall_end",
    contentIndex: idx,
    toolCall: contentBlock,
    partial: output,
  });
}
```

### Writing Control Response to Stdin

```typescript
// Source: Wire protocol (verified via SDK docs + Substack article)

function writeControlResponse(
  proc: ChildProcess,
  requestId: string,
  behavior: "allow" | "deny",
  message?: string,
): void {
  const response = {
    type: "control_response",
    request_id: requestId,
    response: {
      subtype: "success",
      response: {
        behavior,
        ...(message ? { message } : {}),
      },
    },
  };
  proc.stdin!.write(JSON.stringify(response) + "\n");
}
```

## State of the Art

| Old Approach                          | Current Approach                             | When Changed                        | Impact                                         |
| ------------------------------------- | -------------------------------------------- | ----------------------------------- | ---------------------------------------------- |
| SDK `canUseTool` callback             | Direct CLI control protocol via stdin/stdout | Project-specific (we don't use SDK) | Must implement wire protocol manually          |
| Separate lookup tables per direction  | Single-source mapping table                  | Project decision                    | Derive both directions from one definition     |
| Drop unknown tool args in translation | Pass through unknown args unchanged          | Project decision (differs from ref) | More robust; ref project's approach loses args |

**Deprecated/outdated:**

- Reference project's `SDK_TO_PI_TOOL_NAME` / `PI_TO_SDK_TOOL_NAME` as separate tables: replaced by single-source approach per project decision

## Open Questions

1. **Exact control_response wire format nesting**
   - What we know: The format uses `{ type: "control_response", request_id, response: { subtype: "success", response: { behavior, message } } }` based on the SDK architecture article
   - What's unclear: The exact nesting (single vs double `response` wrapping) could not be verified from official CLI docs since they don't document the raw wire format
   - Recommendation: Implement the documented format and test against a real CLI subprocess. If responses fail, try the simpler flat format `{ type: "control_response", request_id, behavior, message }` as a fallback. Add a debug log for control protocol messages to aid troubleshooting.

2. **Does message_stop emit when tool calls are denied?**
   - What we know: Reference project checks `sawToolCall` and overrides `stopReason` to "toolUse" at `message_stop`. But in the reference project, tools are denied via SDK callback, not via wire protocol -- the CLI behavior after receiving a deny response is not documented.
   - What's unclear: After we deny tool execution, does the CLI send `content_block_stop` + `message_stop`? Or does it send a different event sequence?
   - Recommendation: The CLI should still complete the message normally (content_block_stop, message_delta, message_stop) since it already streamed the tool_use content block. Pi needs the full ToolCall to execute the tool on its side. The message_stop handler should check for tool_use content and set stopReason = "toolUse".

## Validation Architecture

### Test Framework

| Property           | Value                               |
| ------------------ | ----------------------------------- |
| Framework          | vitest ^3.0.0                       |
| Config file        | `vitest.config.ts`                  |
| Quick run command  | `npx vitest run --reporter=verbose` |
| Full suite command | `npx vitest run --reporter=verbose` |

### Phase Requirements -> Test Map

| Req ID       | Behavior                                             | Test Type | Automated Command                                                              | File Exists? |
| ------------ | ---------------------------------------------------- | --------- | ------------------------------------------------------------------------------ | ------------ |
| STRM-03      | Control_request detection + control_response writing | unit      | `npx vitest run tests/control-handler.test.ts -t "control" --reporter=verbose` | No -- Wave 0 |
| TOOL-01      | Deny built-in tool execution via control protocol    | unit      | `npx vitest run tests/control-handler.test.ts -t "deny" --reporter=verbose`    | No -- Wave 0 |
| TOOL-02      | Allow MCP-prefixed tool execution                    | unit      | `npx vitest run tests/control-handler.test.ts -t "mcp" --reporter=verbose`     | No -- Wave 0 |
| TOOL-03      | Bidirectional tool name mapping                      | unit      | `npx vitest run tests/tool-mapping.test.ts -t "name" --reporter=verbose`       | No -- Wave 0 |
| TOOL-04      | Bidirectional argument translation                   | unit      | `npx vitest run tests/tool-mapping.test.ts -t "arg" --reporter=verbose`        | No -- Wave 0 |
| STRM-03+TOOL | Integration: full tool call through event bridge     | unit      | `npx vitest run tests/event-bridge.test.ts -t "tool_use" --reporter=verbose`   | No -- Wave 0 |

### Sampling Rate

- **Per task commit:** `npx vitest run --reporter=verbose`
- **Per wave merge:** `npx vitest run --reporter=verbose`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps

- [ ] `tests/tool-mapping.test.ts` -- covers TOOL-03, TOOL-04 (tool name + arg mapping)
- [ ] `tests/control-handler.test.ts` -- covers STRM-03, TOOL-01, TOOL-02 (control protocol)
- [ ] New test cases in `tests/event-bridge.test.ts` -- covers tool_use content block streaming (toolcall_start/delta/end events)
- [ ] New test cases in `tests/provider.test.ts` -- covers control_request handling in the stream processing loop

## Sources

### Primary (HIGH confidence)

- Reference project `claude-agent-sdk-pi` (local clone at `/tmp/claude-agent-sdk-pi/index.ts`) -- tool mapping tables (lines 12-29), argument mapping (lines 374-421), event bridge tool_use handling (lines 700-796), parsePartialJson (lines 542-549), deny constant (line 34)
- `@mariozechner/pi-ai` type definitions (`node_modules/@mariozechner/pi-ai/dist/types.d.ts`) -- ToolCall interface, toolcall_start/delta/end event types, AssistantMessage content union type
- Claude Agent SDK TypeScript docs (https://platform.claude.com/docs/en/agent-sdk/typescript) -- PermissionResult type with allow/deny behavior, CanUseTool callback signature
- Pi-ai README (`node_modules/@mariozechner/pi-ai/README.md`) -- toolcall event documentation with streaming examples

### Secondary (MEDIUM confidence)

- Substack article "Inside the Claude Agent SDK" (https://buildwithaws.substack.com/p/inside-the-claude-agent-sdk-from) -- control_request/control_response wire format with request_id and nested response structure
- Claude API streaming docs (https://platform.claude.com/docs/en/build-with-claude/streaming) -- input_json_delta event type, partial_json field for tool argument streaming

### Tertiary (LOW confidence)

- Exact nesting of control_response wire format -- verified from architectural article, not official CLI docs. The format `{ response: { subtype: "success", response: { behavior } } }` has double nesting that needs runtime validation.

## Metadata

**Confidence breakdown:**

- Standard stack: HIGH -- no new dependencies, using existing project patterns
- Architecture: HIGH -- reference project provides complete working example, adapted for CLI control protocol
- Tool mapping: HIGH -- exact tables extracted from reference project, verified against pi-ai types
- Control protocol: MEDIUM -- wire format verified from architectural analysis, not official CLI documentation. Double-nesting structure needs runtime validation.
- Pitfalls: HIGH -- identified through reference project analysis and existing codebase review

**Research date:** 2026-03-13
**Valid until:** 2026-04-13 (30 days -- stable domain, mapping tables unlikely to change)
