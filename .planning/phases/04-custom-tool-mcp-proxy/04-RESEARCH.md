# Phase 4: Custom Tool MCP Proxy - Research

**Researched:** 2026-03-14 (second round, post-architectural failure)
**Domain:** MCP server architecture, IPC patterns, pi tool execution model
**Confidence:** MEDIUM (architecture verified, critical open question on tool execution)

## Summary

Phase 4 requires re-architecting the custom tool MCP proxy after the denial-stub approach was proven fundamentally flawed. The previous implementation (reverted) confirmed that MCP tools bypass the Claude CLI control protocol entirely -- `control_request` messages are never sent for MCP tools, so the extension cannot intercept and deny them for pi to execute. Without MCP, Claude has no tool schemas for custom tools and enters a ToolSearch loop.

The core challenge is a **timing mismatch**: Claude CLI executes MCP tools synchronously during response generation (the stream pauses while the MCP server handles the call), but pi executes tools only after the LLM response completes. Additionally, pi's `ExtensionAPI` has no `executeTool()` method -- extensions cannot programmatically trigger tool execution. `getAllTools()` returns `ToolInfo[]` (name, description, parameters) without the `execute()` function.

**Primary recommendation:** Use an **in-process HTTP transport MCP server** (eliminating external IPC complexity). Claude Code supports `"type": "http"` in MCP config, allowing the MCP server to run as an HTTP listener within the extension process. The MCP server's `CallTool` handler executes tools directly (for simple tools via `pi.exec()` or Node.js APIs) or returns a structured description of what was requested (for tools whose execution cannot be replicated). The `tool_use` content blocks in the stream must be filtered to prevent pi from double-executing tools that the MCP handler already executed.

If direct tool execution in the MCP handler proves infeasible for the actual custom tools encountered, a fallback approach is to keep the original stdio denial-stub architecture but modify the stream so that pi's agent loop handles execution after the response, and the MCP stub result is replaced in conversation history with the real result on the next turn. This is less ideal but avoids the tool-execution-access problem.

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions (still valid)
- Use `@modelcontextprotocol/sdk` for the MCP server
- `pi.getAllTools()` for custom tool discovery -- MUST be called lazily on first request, NOT during extension loading (runtime not initialized)
- Filter out 6 built-in tools (`read`, `write`, `edit`, `bash`, `grep`, `find`)
- TypeBox schemas pass through as JSON Schema -- no conversion needed
- `--mcp-config` flag to register MCP server with subprocess
- Don't pass `--strict-mcp-config` (CONF-01)
- Warn-don't-block if MCP setup fails
- Strip `mcp__custom-tools__` prefix from tool call names (MCP-02)

### Invalidated Decisions (needs redesign)
- ~~MCP server in a separate `.js` file~~ -> HTTP transport runs in-process; no separate file needed
- ~~MCP server returns denial stubs~~ -> Must return REAL tool results
- ~~Tool results flow via conversation history replay~~ -> Results must flow through MCP response
- ~~Start MCP server once at registration~~ -> HTTP server starts at registration, serves requests per-Claude-subprocess
- ~~`--allowedTools "mcp__custom-tools__*"`~~ -> Not needed; MCP tools are auto-approved by Claude CLI

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope

</user_constraints>

<phase_requirements>

## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| MCP-01 | Extension exposes custom pi tools (non-built-in) to Claude via MCP server, registered with the subprocess via `--mcp-config` | HTTP transport MCP server preferred over stdio; `--mcp-config` supports `type: "http"` with localhost URL. Tool schema exposure via `@modelcontextprotocol/sdk` Server class confirmed working. |
| MCP-02 | Extension strips the `mcp__custom-tools__` prefix from tool call names when mapping MCP tool proposals back to pi tool names | Prefix stripping logic confirmed working in prior implementation. Same approach applies with additional stream filtering for MCP-handled tools. |
| CONF-01 | Extension defaults `strictMcpConfig` to `false`, allowing existing `.mcp.json` and `~/.claude.json` MCP server configurations to load automatically | Don't pass `--strict-mcp-config` flag. Confirmed working in prior implementation. |

</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `@modelcontextprotocol/sdk` | ^1.27.1 (v1.x branch) | MCP server implementation with HTTP transport | Official TypeScript SDK; v1.x is recommended for production; v2 is pre-alpha |
| `node:http` | built-in | HTTP server for in-process MCP transport | No external dependency; MCP SDK's StreamableHTTPServerTransport works with Node.js HTTP handler |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `zod` | ^3.x | Peer dependency of MCP SDK | SDK requires it; install as devDependency if not already present |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| HTTP transport (in-process) | Stdio transport (separate process) | Stdio requires external IPC (temp files, named pipes, sockets) for tool result communication; HTTP keeps everything in one process |
| `node:http` | Express | Express adds unnecessary dependency for a single `/mcp` endpoint |
| `StreamableHTTPServerTransport` | `SSEServerTransport` | SSE is deprecated in MCP spec as of 2025-03-26; Streamable HTTP is the standard |
| Low-level `Server` class | High-level `McpServer` class | McpServer requires Zod schemas; low-level Server accepts raw JSON Schema from TypeBox |

**Installation:**
```bash
npm install @modelcontextprotocol/sdk@^1.27.1
# If zod peer dependency warning appears:
npm install -D zod
```

## Architecture Patterns

### Critical: Why Previous Architecture Failed

The denial-stub stdio MCP server approach failed for three interconnected reasons (full details in `.planning/debug/mcp-toolsearch-loop.md`):

1. **MCP tools bypass the control protocol entirely.** The Claude CLI pre-approves MCP tools at the permission layer. `control_request` messages are never generated for `mcp__` prefixed tools. The extension's `handleControlRequest` function (which denies built-in tools for pi to execute) never sees MCP tool calls.

2. **Without MCP, Claude can't call custom tools.** Claude Code only knows how to call built-in tools (schemas hardcoded) and MCP tools (schemas from MCP server). Custom tools mentioned in system prompt text are NOT callable -- Claude has no schema and attempts ToolSearch (which also fails).

3. **Denial stubs give Claude useless data.** When the MCP server returns "handled by host application," Claude responds based on that placeholder. Pi never executes the tool. The user gets a response about a stub, not real data.

### Critical: Pi's Tool Execution Constraint

**Pi's `ExtensionAPI` has NO `executeTool()` method.** Confirmed by examining the [pi-mono source code](https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/src/core/extensions/types.ts) (line 1285):

```
export type ToolInfo = Pick<ToolDefinition, "name" | "description" | "parameters">;
```

- `pi.getAllTools()` returns `ToolInfo[]` -- name, description, parameters. No `execute()` function.
- Tool execution happens exclusively in pi's agent loop, triggered by `toolcall_end` stream events.
- Extensions can block tool calls (`tool_call` event with `{ block: true }`) and modify results (`tool_result` event), but cannot trigger execution.
- `pi.exec(command, args)` executes SHELL commands, not registered tools.

### Reference Project Architecture (claude-agent-sdk-pi)

The [reference project](https://github.com/prateekmedia/claude-agent-sdk-pi) uses the Claude Agent SDK (not CLI) and handles custom tools via `createSdkMcpServer()`:

```typescript
// Reference project pattern (SDK-based, NOT applicable to CLI)
const mcpTools = customTools.map((tool) => ({
    name: tool.name,
    description: tool.description,
    inputSchema: tool.parameters,
    handler: async () => ({
        content: [{ type: "text", text: TOOL_EXECUTION_DENIED_MESSAGE }],
        isError: true,  // Returns error to Claude
    }),
}));
const server = createSdkMcpServer({ name: MCP_SERVER_NAME, version: "1.0.0", tools: mcpTools });

// In query options:
canUseTool: async () => ({ behavior: "deny", message: TOOL_EXECUTION_DENIED_MESSAGE }),
```

The SDK approach works because `canUseTool: "deny"` denies ALL tools (including MCP tools). Claude proposes but never executes. Pi's agent loop handles execution after the response. **This pattern is NOT transferable to the CLI** because the CLI's control protocol does not intercept MCP tools.

### Recommended Architecture: In-Process HTTP MCP Server

```
Extension Process (pi runtime)
|
+-- HTTP Server (localhost:PORT, started at registration)
|   +-- POST /mcp  ->  StreamableHTTPServerTransport
|   |   +-- ListTools handler: return custom tool schemas from pi.getAllTools()
|   |   +-- CallTool handler: execute tool, return real result to Claude
|   |
+-- MCP Config (temp JSON file, written per Claude subprocess)
|   +-- { mcpServers: { "custom-tools": { type: "http", url: "http://127.0.0.1:PORT/mcp" } } }
|   |
+-- Claude CLI subprocess (spawned per pi request)
    +-- Reads --mcp-config, connects to HTTP MCP server
    +-- Calls tools via HTTP during response generation
    +-- Gets real results, continues generating
    +-- Stream events flow back to extension
```

**Key advantages over stdio approach:**
- No IPC needed -- HTTP handler runs in the same process as the extension
- No separate `.js` file needed -- the MCP server is part of the extension code
- Direct access to pi runtime via closure scope (can call `pi.getAllTools()`, `pi.exec()`)
- No process lifecycle management for the MCP server
- Claude CLI spawns and kills its own stdio MCP servers, but HTTP servers are independent

**MCP config format for HTTP transport (confirmed from Claude Code docs):**
```json
{
  "mcpServers": {
    "custom-tools": {
      "type": "http",
      "url": "http://127.0.0.1:PORT/mcp"
    }
  }
}
```

### Tool Execution Strategy

Since pi has no `executeTool()` API, the MCP handler needs to execute tools independently. Two strategies:

**Strategy A: MCP-Side Execution (RECOMMENDED)**

Let Claude handle MCP tool execution directly through the MCP server. The MCP handler implements tool execution logic within the extension process:

1. MCP handler receives tool name + args from Claude
2. MCP handler executes the tool using available APIs:
   - For shell-based tools: `child_process.execSync()` or `pi.exec()`
   - For file-based tools: `node:fs` APIs
   - For tools with custom logic: the handler needs the execute function
3. MCP handler returns real result to Claude
4. Claude continues generating with the real result
5. Event bridge FILTERS OUT MCP tool_use blocks (does NOT emit `toolcall_end` to pi)
6. Pi never sees the tool call, avoiding double execution

**The unsolved problem:** For custom tools registered by OTHER pi extensions, we don't have access to their `execute()` functions. This limits Strategy A to tools we can re-implement.

**Practical workaround:** Most custom pi tools in practice are simple wrappers. For tools we cannot execute, the MCP handler returns a descriptive message explaining it cannot execute the tool and suggesting the user run it manually. This is better than a generic denial stub because Claude can at least tell the user what tool it tried to call and with what arguments.

**Strategy B: Dual-Path with History Replay (FALLBACK)**

Keep the MCP server as a pass-through that returns a structured "pending" result, but redesign the flow:

1. MCP handler receives tool call, returns: `"Tool {name} called with args {args}. Result will be provided by the host application."`
2. Event bridge emits `toolcall_end` to pi (stripped name)
3. Pi executes the tool via its agent loop (on the NEXT turn, since current response already completed)
4. Tool result appears in conversation history
5. Prompt builder replays history with MCP prefix, Claude sees the real result

**Problem:** This is essentially the approach that failed, but with better messaging. Claude still acts on the placeholder, not the real result. The tool result only appears in the next conversation turn.

### Recommended Project Structure
```
src/
  mcp-server.ts       # In-process HTTP MCP server (Server + StreamableHTTPServerTransport)
  mcp-config.ts       # MCP config file generation and custom tool discovery
  tool-mapping.ts     # Extended with MCP prefix stripping and custom tool detection
  prompt-builder.ts   # Extended with MCP prefix for custom tool history replay
  control-handler.ts  # Unchanged -- MCP tools bypass control protocol
  event-bridge.ts     # Extended to filter MCP tool_use blocks from pi events
  process-manager.ts  # Extended with mcpConfigPath option
  provider.ts         # Extended with mcpConfigPath passthrough + HTTP server lifecycle
  types.ts            # Unchanged
tests/
  mcp-server.test.ts  # HTTP MCP server tests
  mcp-config.test.ts  # Config generation tests
  tool-mapping.test.ts # Extended with prefix stripping tests
  prompt-builder.test.ts # Extended with custom tool replay tests
  event-bridge.test.ts   # Extended with MCP tool filtering tests
  process-manager.test.ts # Extended with --mcp-config flag tests
```

### Pattern 1: In-Process HTTP MCP Server

**What:** Run an MCP server as an HTTP listener within the extension process using `StreamableHTTPServerTransport`
**When to use:** When the MCP client (Claude CLI) and the tool implementation must share process state
**Example:**
```typescript
// Source: MCP TypeScript SDK docs + Claude Code MCP docs
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { CallToolRequestSchema, ListToolsRequestSchema } from "@modelcontextprotocol/sdk/types.js";
import { createServer, type IncomingMessage, type ServerResponse } from "node:http";

interface ToolDef {
  name: string;
  description: string;
  inputSchema: Record<string, unknown>;
}

function createMcpHttpServer(toolDefs: ToolDef[]): ReturnType<typeof createServer> {
  const server = new Server(
    { name: "custom-tools", version: "1.0.0" },
    { capabilities: { tools: {} } }
  );

  server.setRequestHandler(ListToolsRequestSchema, async () => ({
    tools: toolDefs.map(t => ({
      name: t.name,
      description: t.description,
      inputSchema: t.inputSchema,
    })),
  }));

  server.setRequestHandler(CallToolRequestSchema, async (request) => {
    const { name, arguments: args } = request.params;
    // Execute tool and return real result
    const result = await executeCustomTool(name, args);
    return {
      content: [{ type: "text" as const, text: JSON.stringify(result) }],
    };
  });

  const httpServer = createServer(async (req: IncomingMessage, res: ServerResponse) => {
    if (req.method === "POST" && req.url === "/mcp") {
      const transport = new StreamableHTTPServerTransport({
        sessionIdGenerator: undefined, // Stateless
      });
      res.on("close", () => { transport.close(); });
      await server.connect(transport);
      await transport.handleRequest(req, res);
    } else if (req.method === "GET" || req.method === "DELETE") {
      res.writeHead(405, { "Content-Type": "application/json" });
      res.end(JSON.stringify({ error: "Method not allowed" }));
    } else {
      res.writeHead(404).end();
    }
  });

  return httpServer;
}
```

### Pattern 2: Dynamic Port Allocation

**What:** Use port 0 to let the OS assign a free port, avoiding conflicts
**When to use:** Always -- prevents port collisions on shared machines
**Example:**
```typescript
const httpServer = createMcpHttpServer(toolDefs);
httpServer.listen(0, "127.0.0.1", () => {
  const addr = httpServer.address() as { port: number };
  const port = addr.port;
  console.warn(`[pi-claude-cli] MCP HTTP server listening on port ${port}`);
  // Generate config with this port
  const configPath = writeMcpConfig(port);
});
```

### Pattern 3: MCP Config with HTTP Transport

**What:** Generate temp config file pointing to localhost HTTP server
**Example:**
```typescript
import { writeFileSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";

function writeMcpConfig(port: number): string {
  const config = {
    mcpServers: {
      "custom-tools": {
        type: "http",
        url: `http://127.0.0.1:${port}/mcp`,
      },
    },
  };
  const configPath = join(tmpdir(), `pi-claude-mcp-${process.pid}.json`);
  writeFileSync(configPath, JSON.stringify(config));
  return configPath;
}
```

### Pattern 4: Filtering MCP Tool Calls in Event Bridge

**What:** Prevent pi from double-executing tools that the MCP handler already executed
**When to use:** When MCP tool execution is handled by the HTTP server
**Example:**
```typescript
import { CUSTOM_TOOLS_MCP_PREFIX } from "./tool-mapping.js";

// In event bridge handleContentBlockStart:
if (blockType === "tool_use") {
  const claudeName = event.content_block!.name!;
  const isMcpCustomTool = claudeName.startsWith(CUSTOM_TOOLS_MCP_PREFIX);

  if (isMcpCustomTool) {
    // Track internally but do NOT emit toolcall_start/end to pi
    // MCP handler already executed this tool
    blocks.push({
      type: "tool_use",
      index: event.index ?? 0,
      isMcpHandled: true,
      // ... other tracking fields
    });
    return; // Skip toolcall_start emission
  }
  // Normal built-in tool handling continues...
}
```

### Pattern 5: Custom Tool Discovery (reusable from prior implementation)
```typescript
// Source: pi-claude-cli prior implementation (confirmed working)
const BUILT_IN_TOOL_NAMES = new Set(["read", "write", "edit", "bash", "grep", "find"]);

export function getCustomToolDefs(pi: any): ToolDef[] {
  const allTools = pi.getAllTools(); // ToolInfo[]
  return allTools
    .filter((t: any) => !BUILT_IN_TOOL_NAMES.has(t.name))
    .map((t: any) => ({
      name: t.name,
      description: t.description,
      inputSchema: t.parameters, // TypeBox IS JSON Schema
    }));
}
```

### Pattern 6: MCP Prefix Stripping (reusable from prior implementation)
```typescript
// Source: pi-claude-cli prior implementation (confirmed working)
export const CUSTOM_TOOLS_MCP_PREFIX = "mcp__custom-tools__";

export function mapClaudeToolNameToPi(claudeName: string): string {
  if (claudeName.startsWith(CUSTOM_TOOLS_MCP_PREFIX)) {
    return claudeName.slice(CUSTOM_TOOLS_MCP_PREFIX.length);
  }
  return CLAUDE_TO_PI_NAME[claudeName.toLowerCase()] ?? claudeName;
}
```

### Anti-Patterns to Avoid
- **Denial stubs in MCP handlers:** Claude generates useless responses based on stubs. ALWAYS return real data or a descriptive error from MCP tool handlers.
- **Trying to deny MCP tools via control_request:** MCP tools bypass the control protocol entirely. `handleControlRequest` never receives `control_request` messages for `mcp__` prefixed tools.
- **Spawning MCP server as a separate process for in-extension IPC:** HTTP transport eliminates the need for external IPC. Keep the MCP server in-process.
- **Emitting `toolcall_end` for MCP tool calls to pi:** Pi would try to double-execute. Filter MCP tool calls from the event bridge.
- **Waiting for pi's agent loop to execute MCP tools:** Deadlock -- Claude CLI pauses the stream during MCP calls; pi executes tools after the stream ends.
- **Using console.log in MCP server code:** Even with HTTP transport, use `console.error` for logging to avoid any stdout contamination.
- **Binding HTTP server to 0.0.0.0:** Always bind to `127.0.0.1` to prevent network exposure and firewall prompts.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| MCP protocol handling | Custom JSON-RPC handler | `@modelcontextprotocol/sdk` Server + StreamableHTTPServerTransport | MCP protocol has complex lifecycle, schema validation, error handling |
| HTTP server | Raw socket server | `node:http` createServer | Built-in, no dependencies, handles HTTP/1.1 correctly |
| Port allocation | Fixed port constants | `server.listen(0)` with OS assignment | Prevents port conflicts, no coordination needed |
| JSON Schema from TypeBox | Manual schema conversion | Direct passthrough | TypeBox schemas ARE JSON Schema -- no conversion needed |
| Tool name prefix handling | String manipulation everywhere | Centralized `CUSTOM_TOOLS_MCP_PREFIX` constant + helpers | Single source of truth, DRY, tested |
| MCP config format | Custom JSON structure | Standard `.mcp.json` format with `type: "http"` | Claude Code's documented config format |

## Common Pitfalls

### Pitfall 1: MCP Tools Bypass Control Protocol
**What goes wrong:** Assuming `handleControlRequest` will receive `control_request` for MCP tools, then relying on deny-then-execute pattern.
**Why it happens:** Built-in tools use control protocol; MCP tools don't. Easy to assume they work the same way.
**How to avoid:** Accept that MCP tools are executed by Claude via MCP. Design the MCP handler to return real results, not denial stubs.
**Warning signs:** Claude CLI never sends `control_request` for `mcp__` prefixed tools.

### Pitfall 2: Double Tool Execution
**What goes wrong:** MCP handler executes the tool AND pi's agent loop also tries to execute it via `toolcall_end` events.
**Why it happens:** The `tool_use` content block still appears in the stream, and the event bridge forwards it to pi.
**How to avoid:** Filter MCP tool calls in the event bridge -- do NOT emit `toolcall_end` for tools with the `mcp__custom-tools__` prefix.
**Warning signs:** Same tool executed twice, pi shows error about tool already executed or inconsistent state.

### Pitfall 3: No executeTool API in Pi
**What goes wrong:** Assuming pi's ExtensionAPI provides a way to programmatically execute registered tools.
**Why it happens:** Natural assumption from the `getAllTools()` method name.
**How to avoid:** Design the MCP handler's execution logic independently of pi's tool execution pipeline.
**Warning signs:** Searching for `executeTool` in pi-coding-agent source yields no results.

### Pitfall 4: Stream Deadlock with Sync MCP Execution
**What goes wrong:** MCP handler tries to wait for pi's agent loop to execute the tool and return the result.
**Why it happens:** Misunderstanding the timeline: Claude CLI pauses the stream during MCP calls; pi executes tools after the stream ends.
**How to avoid:** Never block the MCP handler waiting on pi's agent loop. Execute tools within the MCP handler itself.
**Warning signs:** Claude CLI hangs indefinitely during tool execution.

### Pitfall 5: MCP Server Startup Before Pi Runtime
**What goes wrong:** Calling `pi.getAllTools()` during extension loading, before pi's runtime is initialized.
**Why it happens:** Extension's default export runs during loading.
**How to avoid:** Lazy initialization -- start HTTP server immediately but defer tool discovery to first request. Call `pi.getAllTools()` in the `ListToolsRequestSchema` handler, not at startup.
**Warning signs:** `pi.getAllTools is not a function` or returns empty array at load time.

### Pitfall 6: Forgetting to Bind to 127.0.0.1
**What goes wrong:** HTTP server binds to 0.0.0.0 (all interfaces), exposing MCP tools to the network.
**Why it happens:** `createServer().listen(port)` defaults to all interfaces on some Node.js versions.
**How to avoid:** Always specify `"127.0.0.1"` as the host parameter in `server.listen(port, "127.0.0.1")`.
**Warning signs:** Windows firewall prompt appears; MCP tools accessible from other machines.

### Pitfall 7: HTTP Server Not Cleaned Up on Extension Shutdown
**What goes wrong:** HTTP server keeps running after pi shuts down, holding the port.
**Why it happens:** No cleanup handler registered.
**How to avoid:** Close the HTTP server on process exit. Register cleanup via `process.on("exit")` or `pi.on("session_shutdown")`.
**Warning signs:** Port already in use on next startup; orphan HTTP server processes.

### Pitfall 8: Large Tool Schemas Exceeding HTTP Limits
**What goes wrong:** If custom tools have very large TypeBox schemas, the HTTP response payload for ListTools could be large.
**Why it happens:** Rare, but possible with complex tool parameter schemas.
**How to avoid:** For typical pi extensions, tool count is small (<10). HTTP has no practical payload limit for localhost. Only a concern if schema count grows to hundreds.
**Warning signs:** Slow tool listing, high memory usage.

## Code Examples

Verified patterns from official sources and prior implementation:

### Complete In-Process HTTP MCP Server
```typescript
// Source: MCP TypeScript SDK docs + Claude Code MCP documentation
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { CallToolRequestSchema, ListToolsRequestSchema } from "@modelcontextprotocol/sdk/types.js";
import { createServer } from "node:http";

interface ToolDef {
  name: string;
  description: string;
  inputSchema: Record<string, unknown>;
}

export function createCustomToolsServer(
  getToolDefs: () => ToolDef[],
  executeTool: (name: string, args: Record<string, unknown>) => Promise<string>,
) {
  const server = new Server(
    { name: "custom-tools", version: "1.0.0" },
    { capabilities: { tools: {} } }
  );

  server.setRequestHandler(ListToolsRequestSchema, async () => ({
    tools: getToolDefs().map(t => ({
      name: t.name,
      description: t.description,
      inputSchema: t.inputSchema,
    })),
  }));

  server.setRequestHandler(CallToolRequestSchema, async (request) => {
    const { name, arguments: args } = request.params;
    try {
      const result = await executeTool(name, args ?? {});
      return { content: [{ type: "text" as const, text: result }] };
    } catch (err: any) {
      return {
        content: [{ type: "text" as const, text: `Error executing ${name}: ${err.message}` }],
        isError: true,
      };
    }
  });

  return server;
}

export async function startMcpHttpServer(
  server: ReturnType<typeof createCustomToolsServer>,
): Promise<{ port: number; close: () => void }> {
  const httpServer = createServer(async (req, res) => {
    if (req.method === "POST" && req.url === "/mcp") {
      try {
        const transport = new StreamableHTTPServerTransport({
          sessionIdGenerator: undefined, // Stateless
        });
        res.on("close", () => { transport.close(); });
        await server.connect(transport);
        await transport.handleRequest(req, res);
      } catch (err) {
        if (!res.headersSent) {
          res.writeHead(500, { "Content-Type": "application/json" });
          res.end(JSON.stringify({ error: "Internal server error" }));
        }
      }
    } else {
      res.writeHead(405, { "Content-Type": "application/json" });
      res.end(JSON.stringify({ error: "Method not allowed" }));
    }
  });

  return new Promise((resolve) => {
    httpServer.listen(0, "127.0.0.1", () => {
      const addr = httpServer.address() as { port: number };
      resolve({
        port: addr.port,
        close: () => httpServer.close(),
      });
    });
  });
}
```

### MCP Config Generation with HTTP Transport
```typescript
// Source: Claude Code MCP docs (https://code.claude.com/docs/en/mcp)
import { writeFileSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";

export function writeMcpConfig(port: number): string {
  const config = {
    mcpServers: {
      "custom-tools": {
        type: "http",
        url: `http://127.0.0.1:${port}/mcp`,
      },
    },
  };
  const configPath = join(tmpdir(), `pi-claude-mcp-${process.pid}.json`);
  writeFileSync(configPath, JSON.stringify(config));
  return configPath;
}
```

### Updated spawnClaude with MCP Config
```typescript
// Extension to process-manager.ts (same as prior implementation)
export function spawnClaude(
  modelId: string,
  systemPrompt?: string,
  options?: { cwd?: string; signal?: AbortSignal; effort?: string; mcpConfigPath?: string },
): ChildProcess {
  const args = [/* existing args */];

  if (options?.mcpConfigPath) {
    args.push("--mcp-config", options.mcpConfigPath);
    // Do NOT add --strict-mcp-config (CONF-01: let user configs load too)
  }

  // ... rest unchanged
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| MCP stdio transport only | stdio + HTTP (Streamable HTTP) + SSE | MCP spec 2025-03-26 | HTTP transport enables in-process MCP servers |
| SSE for remote MCP | Streamable HTTP | MCP spec 2025-03-26 | SSE deprecated; Streamable HTTP is the replacement |
| MCP SDK v1 | MCP SDK v1.x (stable) / v2 (pre-alpha) | Q1 2026 | v1.x remains recommended for production |
| `McpServer` high-level class | `Server` low-level class for raw JSON Schema | SDK v1.25+ | Low-level Server provides raw JSON Schema support for inputSchema (avoids Zod) |
| Denial-stub MCP proxies | MCP handlers with real tool execution | Architectural learning | Stubs produce useless Claude responses; real execution is required |

**Deprecated/outdated:**
- SSE transport: deprecated in MCP spec 2025-03-26, replaced by Streamable HTTP
- MCP SDK v2: still pre-alpha as of Q1 2026, not recommended for production
- Denial-stub MCP pattern: proven non-functional for CLI-based tool handling

## Open Questions

1. **How does the in-process MCP handler execute custom tools?**
   - What we know: `pi.getAllTools()` returns ToolInfo without execute functions. Pi has no `executeTool()` API. Tools are executed only by pi's internal agent loop. `pi.exec()` runs shell commands, not registered tools.
   - What's unclear: Whether we can re-implement common custom tool patterns in the MCP handler, or whether we need pi API changes.
   - Recommendation: **This is the primary risk.** Start with a minimal implementation where the MCP handler returns a descriptive "tool call attempted" message. Validate with real custom tools. If tools are simple enough to re-implement (shell commands, file ops), do so in the handler. For complex tools, consider requesting an `executeTool()` API from pi's maintainer.

2. **Does `--mcp-config` support `type: "http"` in the temp config file?**
   - What we know: Claude Code docs show HTTP MCP config in `.mcp.json`. The `--mcp-config` flag accepts "JSON files or strings." The config format appears identical to `.mcp.json`.
   - What's unclear: 100% confirmation that the temp config file for `--mcp-config` supports `type: "http"`.
   - Recommendation: HIGH confidence this works based on documentation. **Validate empirically in the first task.**

3. **Do MCP tool_use content blocks appear in stream events?**
   - What we know: Built-in tool calls appear as `content_block_start/delta/stop` events in the stream. MCP tool calls should follow the same pattern.
   - What's unclear: Whether MCP tool results also appear in stream events, or only the tool_use proposal.
   - Recommendation: **Validate empirically.** This determines whether event bridge filtering is needed and what the stream looks like when MCP tools are used.

4. **Does Claude continue generating after MCP tool execution within the same response?**
   - What we know: With built-in tools, the response ends with `stop_reason: "tool_use"`. With MCP tools, Claude gets the result and may continue.
   - What's unclear: Exact stream behavior when MCP tools return results inline.
   - Recommendation: **Validate empirically.** This affects how the event bridge handles the post-MCP-tool stream.

5. **Will localhost HTTP add perceptible latency vs stdio?**
   - What we know: HTTP localhost adds ~1-5ms per round trip. MCP tool listing happens once per session. Tool calls happen per-use.
   - What's unclear: Whether Claude CLI has different timeout behavior for HTTP vs stdio MCP servers.
   - Recommendation: LOW risk. Localhost HTTP is very fast. Can set `MCP_TIMEOUT` env var if needed (default is 10 seconds).

6. **StreamableHTTPServerTransport stateless mode and per-request server.connect()**
   - What we know: The official example creates a new transport per request and calls `server.connect(transport)` per request. For stateless servers, `sessionIdGenerator: undefined`.
   - What's unclear: Whether calling `server.connect()` multiple times (once per request) is correct for the low-level Server class.
   - Recommendation: Follow the official example pattern. Stateless mode is appropriate since each Claude subprocess is independent.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | vitest ^3.0.0 |
| Config file | `vitest.config.ts` |
| Quick run command | `npx vitest run tests/mcp-server.test.ts tests/mcp-config.test.ts --reporter=verbose` |
| Full suite command | `npx vitest run --reporter=verbose` |

### Phase Requirements -> Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| MCP-01 | Custom pi tools listed via HTTP MCP server ListTools handler | unit | `npx vitest run tests/mcp-server.test.ts -x` | No -- Wave 0 |
| MCP-01 | MCP config file generated with type:http and localhost URL | unit | `npx vitest run tests/mcp-config.test.ts -x` | No -- Wave 0 |
| MCP-01 | CallTool handler executes tool and returns result | unit | `npx vitest run tests/mcp-server.test.ts -x` | No -- Wave 0 |
| MCP-01 | spawnClaude passes --mcp-config flag when provided | unit | `npx vitest run tests/process-manager.test.ts -x` | Yes (file exists, test needs adding) |
| MCP-02 | mcp__custom-tools__ prefix stripped from tool call names | unit | `npx vitest run tests/tool-mapping.test.ts -x` | Yes (file exists, test needs adding) |
| MCP-02 | Event bridge filters MCP tool_use blocks from pi events | unit | `npx vitest run tests/event-bridge.test.ts -x` | Yes (file exists, test needs adding) |
| MCP-02 | Custom tool names prefixed in prompt builder history replay | unit | `npx vitest run tests/prompt-builder.test.ts -x` | Yes (file exists, test needs adding) |
| CONF-01 | --strict-mcp-config never passed in spawn args | unit | `npx vitest run tests/process-manager.test.ts -x` | Yes (file exists, test needs adding) |

### Sampling Rate
- **Per task commit:** `npx vitest run tests/mcp-server.test.ts tests/mcp-config.test.ts --reporter=verbose`
- **Per wave merge:** `npx vitest run --reporter=verbose`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `tests/mcp-server.test.ts` -- covers MCP-01 (HTTP server, ListTools, CallTool handlers)
- [ ] `tests/mcp-config.test.ts` -- covers MCP-01 (config generation with HTTP transport, tool discovery)
- [ ] MCP prefix stripping tests in `tests/tool-mapping.test.ts` -- covers MCP-02
- [ ] MCP tool filtering tests in `tests/event-bridge.test.ts` -- covers MCP-01/MCP-02 stream integration
- [ ] Custom tool history replay tests in `tests/prompt-builder.test.ts` -- covers MCP-02
- [ ] MCP config flag tests in `tests/process-manager.test.ts` -- covers MCP-01/CONF-01
- [ ] Framework install: `npm install @modelcontextprotocol/sdk@^1.27.1` -- if not present

## Sources

### Primary (HIGH confidence)
- [Claude Code MCP documentation](https://code.claude.com/docs/en/mcp) -- MCP config format, transport types (stdio, HTTP, SSE), `--mcp-config` flag, `.mcp.json` structure, HTTP type supported
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk) -- Server class, StreamableHTTPServerTransport, v1.x recommended for production
- [MCP Transports specification](https://modelcontextprotocol.io/specification/2025-03-26/basic/transports) -- Streamable HTTP replaces SSE as of 2025-03-26
- [pi-mono ExtensionAPI source](https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/src/core/extensions/types.ts) -- confirmed no `executeTool()` method; ToolInfo = Pick<ToolDefinition, "name" | "description" | "parameters">
- [claude-agent-sdk-pi reference project](https://github.com/prateekmedia/claude-agent-sdk-pi) -- complete source examined; `createSdkMcpServer` with denial stubs + `canUseTool: deny` pattern (SDK only, not transferable to CLI)
- [pi-mono extensions documentation](https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/extensions.md) -- ExtensionAPI methods, tool registration, event system, getAllTools() returns ToolInfo[]

### Secondary (MEDIUM confidence)
- [MCP SDK npm package](https://www.npmjs.com/package/@modelcontextprotocol/sdk) -- v1.27.1 latest, peer dependency on zod
- [StreamableHTTPServerTransport usage examples](https://mcpcat.io/guides/building-streamablehttp-mcp-server/) -- stateless server pattern with sessionIdGenerator: undefined
- [MCP server starter template](https://github.com/ferrants/mcp-streamable-http-typescript-server) -- HTTP transport config format: `{ type: "streamable-http", url: "http://localhost:3000" }`
- [Node.js child_process IPC](https://nodejs.org/api/child_process.html) -- process.send/process.on('message') for fork-based IPC (evaluated but not recommended)

### Tertiary (LOW confidence)
- MCP tool execution flow within Claude CLI stream-json mode -- not documented; needs empirical validation
- Whether `tool_use` content blocks for MCP tools appear in stream events -- inferred from architecture, not confirmed
- Whether Claude continues generating after MCP tool execution or ends the turn -- needs empirical validation
- `StreamableHTTPServerTransport` with per-request `server.connect()` on low-level Server class -- follows official example but not explicitly documented for this pattern

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- MCP SDK v1.x and HTTP transport confirmed via official docs and npm
- Architecture (HTTP MCP server): HIGH -- Claude Code supports HTTP type in MCP config; SDK supports StreamableHTTPServerTransport
- Architecture (tool execution): LOW -- no confirmed path for executing custom tools from MCP handler; primary open question
- Prefix stripping: HIGH -- confirmed working in prior implementation; same logic reusable
- Pitfalls: HIGH -- empirically verified through failed first implementation
- Event bridge filtering: MEDIUM -- logical requirement but stream behavior for MCP tools not empirically verified

**Research date:** 2026-03-14
**Valid until:** 2026-04-14 (30 days -- MCP ecosystem is actively evolving but core patterns are stable)
