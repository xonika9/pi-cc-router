# Technology Stack

**Project:** pi-claude-cli
**Researched:** 2026-03-13
**Overall Confidence:** MEDIUM-HIGH

## Constraints from PROJECT.md

Before diving into recommendations, these constraints are non-negotiable:

- **No Claude Agent SDK** -- The project explicitly replaces `@anthropic-ai/claude-agent-sdk` with a direct CLI subprocess
- **No direct Anthropic API calls** -- Authentication is via Claude CLI's Pro/Max subscription
- **Transport:** `claude -p --input-format stream-json --output-format stream-json --verbose` subprocess only
- **Runtime:** pi extensions run in Node.js via `jiti` (runtime TypeScript loader) -- no build step required
- **Platform:** Must work on Windows, macOS, and Linux

## Recommended Stack

### Extension Runtime (provided by pi)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Node.js | 22+ | Runtime | pi coding agent requires Node.js 22+. Extensions run within pi's process. |
| TypeScript | 5.x | Language | Pi loads `.ts` files directly via `jiti` -- no compilation step needed. Write TypeScript, ship TypeScript. |
| jiti | 2.x (bundled by pi) | TS loader | Pi uses jiti internally to load extensions at runtime. You don't install this -- pi provides it. Extensions can use ESM `import` syntax and TypeScript natively. |

**Confidence:** HIGH -- Verified via pi-mono docs and npm package metadata.

**Key implication:** No `tsconfig.json` build step is needed for distribution. The extension ships as `.ts` source files. However, you should still have a `tsconfig.json` for IDE type checking and a `package.json` for dependency management.

### Subprocess Management

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| `node:child_process` (spawn) | Built-in | Spawn `claude` CLI | Native Node.js API. Spawns the `claude -p` subprocess with stdin/stdout streaming. No external dependency needed. |
| `cross-spawn` | ^7.0.6 | Cross-platform spawn | Drop-in replacement for `child_process.spawn()` that handles Windows `.cmd` shim resolution. On Windows, `claude` resolves to `claude.cmd` -- native `spawn` breaks without `shell: true`, but `cross-spawn` handles this correctly without the security implications of shell mode. |

**Confidence:** HIGH -- `cross-spawn` is the established solution (used by virtually every cross-platform Node.js CLI tool). Native `spawn` works on macOS/Linux but fails on Windows for `.cmd`-shimmed binaries like `claude`.

**Decision rationale:** Use `cross-spawn` instead of `shell: true` because shell mode introduces command injection risk and changes argument quoting behavior. `cross-spawn` solves the Windows problem without those tradeoffs.

### NDJSON Stream Parsing

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| `node:readline` | Built-in | Line-by-line stdout parsing | Use `readline.createInterface({ input: childProcess.stdout, terminal: false })` to split the NDJSON stream into individual lines. Built-in, zero-dependency, handles buffering and partial lines correctly. |
| `JSON.parse` | Built-in | Parse individual JSON lines | Each line from the `claude` subprocess is a complete JSON object. No streaming JSON parser needed -- just `JSON.parse()` each line. |

**Confidence:** HIGH -- This is the standard Node.js pattern for NDJSON. No external library needed.

**What NOT to use:**
- `ndjson` npm package -- Last published 6 years ago. Abandoned.
- `stream-json` npm package -- Overkill. This is for parsing a single large JSON document incrementally. Our use case is NDJSON (complete JSON per line), which is simpler.
- `split2` -- Works but adds an unnecessary dependency when `readline` does the same thing natively.
- Custom `data` event buffering -- Error-prone. `readline` already handles the `\n`-boundary buffering correctly.

### MCP Server (for Custom Tool Proxy)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| `@modelcontextprotocol/sdk` | ^1.27.1 | MCP server implementation | The official MCP TypeScript SDK. Used to create a stdio MCP server that exposes pi's custom tools to the Claude CLI subprocess. Claude sees them as `mcp__custom-tools__<toolName>`. |
| `zod` | ^3.25.0 | Schema validation | Required peer dependency of `@modelcontextprotocol/sdk`. The MCP SDK imports from `zod/v4` internally but works with zod >=3.25. Use zod for tool input schema definitions. |

**Confidence:** HIGH -- `@modelcontextprotocol/sdk` is the official SDK with 32,000+ dependents. Version 1.27.1 is the latest stable (published Feb 2025). The reference project uses `createSdkMcpServer()` from the Agent SDK for the same purpose; we replicate this with a standalone stdio MCP server.

**Architecture note:** The MCP server runs as a separate stdio subprocess alongside the `claude` process. It is registered with Claude via `--mcp-config` pointing to a temporary JSON config file that declares the server. Claude then connects to it over stdio and discovers the tools.

**What NOT to use:**
- `@anthropic-ai/claude-agent-sdk`'s `createSdkMcpServer()` -- This is the Agent SDK's convenience wrapper. Since we're not using the Agent SDK, we build the MCP server directly with `@modelcontextprotocol/sdk`.
- HTTP/SSE MCP transport -- Overkill for a local tool proxy. Stdio transport is simpler and avoids port conflicts.

### Pi Extension API (consumed, not installed)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| pi extension API | Latest (from pi runtime) | `pi.registerProvider()`, event types | Pi provides the `pi` global object at runtime. You don't install this as a dependency -- it's injected by the pi extension loader. |

**Confidence:** HIGH -- Verified from pi-mono custom-provider docs and extensions docs.

**Key types you'll use (from pi runtime):**

```typescript
// Registering the provider
pi.registerProvider("claude-cli", {
  api: "custom",
  models: [...],  // Model definitions from getModels("anthropic")
  streamSimple: (model, context, options?) => AssistantMessageEventStream
});

// Events emitted by streamSimple
type AssistantMessageEvent =
  | { type: "start" }
  | { type: "text_start" }
  | { type: "text_delta"; text: string }
  | { type: "text_end" }
  | { type: "thinking_start" }
  | { type: "thinking_delta"; text: string }
  | { type: "thinking_end" }
  | { type: "toolcall_start"; id: string; name: string }
  | { type: "toolcall_delta"; text: string }
  | { type: "toolcall_end" }
  | { type: "done" }
  | { type: "error"; error: Error };
```

### Development & Type Checking

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| TypeScript | ~5.7 | Type checking (dev only) | For IDE support and `tsc --noEmit` type checking. Not used for building -- jiti handles runtime loading. |
| `@types/node` | ^22 | Node.js type definitions | Matches the Node.js 22+ requirement. Provides types for `child_process`, `readline`, etc. |

**Confidence:** HIGH

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Subprocess | `cross-spawn` | Native `child_process.spawn` | Native spawn fails on Windows for `.cmd` shims without `shell: true`, which has security implications |
| Subprocess | `cross-spawn` | `execa` | Execa adds significant overhead and ESM-only constraints. `cross-spawn` is lighter and solves the specific Windows problem |
| NDJSON parsing | `node:readline` | `split2` / `ndjson` npm | Built-in readline handles line splitting natively; no need for abandoned or unnecessary packages |
| MCP server | `@modelcontextprotocol/sdk` | Hand-rolled stdio JSON-RPC | The MCP protocol has specific handshake, capability negotiation, and message framing requirements. The official SDK handles all of this correctly |
| MCP server | stdio transport | HTTP/SSE transport | Stdio is simpler for a local subprocess. No port allocation, no network overhead, no CORS |
| Build system | None (jiti) | tsup / esbuild | Pi loads `.ts` directly via jiti. Adding a build step is unnecessary complexity |
| Build system | None (jiti) | tsc compilation | Same reason. Ship `.ts` files. Use `tsc --noEmit` only for type checking during development |
| Schema validation | `zod` | `joi` / `yup` | Zod is required by `@modelcontextprotocol/sdk` (peer dependency). Using anything else means two schema libraries |
| Agent SDK | Direct CLI subprocess | `@anthropic-ai/claude-agent-sdk` | Project constraint: the entire point is to NOT depend on the Agent SDK. Direct subprocess gives full control over the wire protocol |

## Full Dependency List

### Runtime Dependencies

```json
{
  "dependencies": {
    "cross-spawn": "^7.0.6",
    "@modelcontextprotocol/sdk": "^1.27.1",
    "zod": "^3.25.0"
  }
}
```

That's it. Three runtime dependencies. Everything else is either built into Node.js or provided by the pi runtime.

### Dev Dependencies

```json
{
  "devDependencies": {
    "typescript": "~5.7.0",
    "@types/node": "^22.0.0"
  }
}
```

### Package.json Structure

```json
{
  "name": "pi-claude-cli",
  "version": "0.1.0",
  "description": "Pi coding agent extension using Claude Code CLI as LLM backend",
  "license": "MIT",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"]
  },
  "dependencies": {
    "cross-spawn": "^7.0.6",
    "@modelcontextprotocol/sdk": "^1.27.1",
    "zod": "^3.25.0"
  },
  "devDependencies": {
    "typescript": "~5.7.0",
    "@types/node": "^22.0.0"
  }
}
```

**Note on the `pi` manifest:** The `"pi": { "extensions": ["./extensions"] }` field tells pi where to find extension entry points. Without it, pi auto-discovers from conventional directories (`extensions/`, `skills/`, etc.).

## Installation

```bash
# For end users
pi install npm:pi-claude-cli

# For development
git clone <repo>
cd pi-claude-cli
npm install
```

## Critical: What the Claude CLI Subprocess Invocation Looks Like

This is the core of the project -- spawning the `claude` process:

```typescript
import spawn from "cross-spawn";

const claude = spawn("claude", [
  "-p",                              // Non-interactive / print mode
  "--input-format", "stream-json",   // Accept NDJSON on stdin
  "--output-format", "stream-json",  // Emit NDJSON on stdout
  "--verbose",                       // Full turn-by-turn output
  "--include-partial-messages",      // Stream token-by-token events
  "--model", modelId,                // e.g., "claude-sonnet-4-6"
  // MCP config for custom tool proxy:
  "--mcp-config", tmpMcpConfigPath,  // Path to temp JSON file declaring MCP servers
  "--strict-mcp-config",             // Only load MCP servers we declare (optional)
], {
  stdio: ["pipe", "pipe", "pipe"],   // stdin, stdout, stderr all piped
  cwd: workingDirectory,
});
```

The prompt is sent via stdin as a stream-json user message, not via CLI args.

## Wire Protocol Summary (Informs Stack Decisions)

### Output (stdout, NDJSON -- one JSON object per line)

```jsonl
{"type":"stream_event","event":{"type":"message_start","message":{...}}}
{"type":"stream_event","event":{"type":"content_block_start","index":0,"content_block":{"type":"text"}}}
{"type":"stream_event","event":{"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Hello"}}}
{"type":"stream_event","event":{"type":"content_block_stop","index":0}}
{"type":"stream_event","event":{"type":"content_block_start","index":1,"content_block":{"type":"tool_use","id":"toolu_xxx","name":"Read","input":{}}}}
{"type":"stream_event","event":{"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":"{\"file_path\":\"src/index.ts\"}"}}}
{"type":"stream_event","event":{"type":"content_block_stop","index":1}}
{"type":"control_request","request_id":"req_xxx","request":{"subtype":"can_use_tool","tool_name":"Read","tool_use_id":"toolu_xxx","input":{"file_path":"src/index.ts"}}}
```

### Input (stdin, NDJSON -- one JSON object per line)

```jsonl
{"type":"user","message":{"role":"user","content":"Fix the bug in auth.ts"}}
{"type":"control_response","request_id":"req_xxx","response":{"subtype":"success","behavior":"deny"}}
{"type":"control_response","request_id":"req_xxx","response":{"subtype":"success","behavior":"allow"}}
```

This wire protocol is why we need:
- `readline` for line splitting (NDJSON)
- `JSON.parse` for each line
- Type guards to distinguish `stream_event` from `control_request` messages
- A state machine to track content block indices and accumulate partial JSON for tool inputs

## Sources

### Official Documentation (HIGH confidence)
- [Claude Code CLI Reference](https://code.claude.com/docs/en/cli-reference) -- All CLI flags
- [Run Claude Code Programmatically](https://code.claude.com/docs/en/headless) -- Headless mode / Agent SDK CLI usage
- [Agent SDK Streaming Output](https://platform.claude.com/docs/en/agent-sdk/streaming-output) -- Stream event types and structure
- [Agent SDK Permissions](https://platform.claude.com/docs/en/agent-sdk/permissions) -- Permission modes and control protocol
- [Agent SDK Custom Tools](https://platform.claude.com/docs/en/agent-sdk/custom-tools) -- MCP server pattern for custom tools
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk) -- Official MCP SDK, v1.27.1
- [Node.js child_process docs](https://nodejs.org/api/child_process.html) -- spawn API
- [Node.js readline docs](https://nodejs.org/api/readline.html) -- Line-by-line stream parsing

### Pi Ecosystem (HIGH confidence)
- [pi-mono custom-provider docs](https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/custom-provider.md) -- registerProvider API
- [pi-mono extensions docs](https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/extensions.md) -- Extension system, jiti loading
- [pi-mono extension types](https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/src/core/extensions/types.ts) -- TypeScript type definitions
- [claude-agent-sdk-pi](https://github.com/prateekmedia/claude-agent-sdk-pi) -- Reference implementation (MIT)

### NPM Packages (HIGH confidence)
- [cross-spawn](https://www.npmjs.com/package/cross-spawn) -- v7.0.6, cross-platform subprocess spawning
- [@modelcontextprotocol/sdk](https://www.npmjs.com/package/@modelcontextprotocol/sdk) -- v1.27.1, MCP server/client SDK
- [zod](https://www.npmjs.com/package/zod) -- v3.25+/v4.x, schema validation (MCP SDK peer dep)
- [@anthropic-ai/claude-agent-sdk](https://www.npmjs.com/package/@anthropic-ai/claude-agent-sdk) -- v0.2.74 (NOT used, but referenced for protocol understanding)
