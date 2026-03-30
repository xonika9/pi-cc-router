# pi-claude-cli

## What This Is

A pi coding agent extension that acts as a custom LLM provider, routing all LLM calls through the official Claude Code CLI (`claude`) as a subprocess instead of calling the Anthropic API directly. This allows using a Claude Pro/Max subscription for authentication rather than requiring an API key. Adapted from [claude-agent-sdk-pi](https://github.com/prateekmedia/claude-agent-sdk-pi) (MIT), replacing the Claude Agent SDK transport with a direct subprocess using the stream-json wire protocol.

## Core Value

Enable pi users to leverage their Claude Pro/Max subscription as the LLM backend — no API key, no separate billing, full Claude model access through the official CLI.

## Requirements

### Validated

<!-- Shipped and confirmed valuable. -->

(None yet — ship to validate)

### Active

<!-- Current scope. Building toward these. -->

- [ ] Registers as a custom pi provider via `pi.registerProvider()` with all current Claude models
- [ ] Spawns `claude -p --input-format stream-json --output-format stream-json --verbose` as a subprocess per request (stateless — fresh process, full history replayed)
- [ ] Handles bidirectional stream-json control protocol for tool approval/denial
- [ ] Denies built-in tool execution on the Claude side (pi executes tools natively) — EXCEPT `mcp__` prefixed tools, which are allowed through for Claude to execute MCP tool calls
- [ ] Streams tokens back to pi in real-time via `AssistantMessageEventStream` (bridges Claude API stream events to pi's event format)
- [ ] Maps tool names bidirectionally between Claude built-in names (Read, Write, Edit, Bash, Grep, Glob) and pi equivalents (read, write, edit, bash, grep, find)
- [ ] Translates tool arguments between Claude and pi formats (e.g., `file_path` <-> `path`, `old_string` <-> `oldText`)
- [ ] Exposes custom pi tools to Claude via an in-process or stdio MCP server proxy with denied handlers (custom tool MCP proxy)
- [ ] `strictMcpConfig` defaults to `false` so existing `.mcp.json` / `~/.claude.json` MCP configs load automatically
- [ ] Builds prompt from full pi conversation history (flattened message replay with USER/ASSISTANT/TOOL RESULT labels)
- [ ] MIT licensed with attribution to `claude-agent-sdk-pi` in README

### Out of Scope

<!-- Explicit boundaries. Includes reasoning to prevent re-adding. -->

- Persistent subprocess sessions — Adds complexity, unproven with stream-json protocol, stateless matches reference project
- Direct Anthropic API calls — The entire point is avoiding API key requirements
- Claude Agent SDK dependency — Replaced by direct CLI subprocess; SDK is what the reference project uses
- Custom authentication flows — Relies on existing `claude` CLI auth (Pro/Max subscription)

## Context

**Reference project:** [claude-agent-sdk-pi](https://github.com/prateekmedia/claude-agent-sdk-pi) — does the same thing using the Claude Agent SDK as transport. This project replicates its architecture but swaps the SDK for a CLI subprocess.

**Key architectural decisions from research:**

- **Session model:** Stateless, matching the reference project. Each pi LLM request spawns a fresh `claude -p` subprocess with full conversation history replayed as a flattened prompt.
- **Tool denial mechanism:** Uses the stream-json control protocol. CLI sends `control_request` with `subtype: "can_use_tool"` on stdout; extension responds on stdin with `control_response` — `behavior: "deny"` for built-in tools, `behavior: "allow"` for `mcp__` prefixed tools. Experimentally confirmed that CLI flags alone (`--permission-mode plan/dontAsk`, `--tools`, `--disallowedTools`) cannot achieve "propose but don't execute."
- **Streaming:** Bridges Claude API `stream_event` messages (content_block_start/delta/stop, message_start/delta/stop) to pi's `AssistantMessageEventStream` format (text_start/delta/end, toolcall_start/delta/end, thinking_start/delta/end, done/error).
- **MCP proxy for custom tools:** Reference project uses SDK's `createSdkMcpServer()`. This project will need an alternative — likely a stdio MCP server spawned alongside the CLI process, registered via `--mcp-config`.

**Pi provider interface:**

- `pi.registerProvider(name, config)` with `streamSimple` function as the core handler
- Models derived from `getModels("anthropic")` — all current Claude models
- Events: `start`, `text_start/delta/end`, `toolcall_start/delta/end`, `thinking_start/delta/end`, `done`/`error`

**Stream-json wire protocol:**

- Output: NDJSON with `stream_event` wrappers containing raw API events (requires `--verbose` + `--include-partial-messages` for granular streaming)
- Input: NDJSON user messages + `control_response` messages
- Control protocol: `control_request`/`control_response` for tool approval, with `can_use_tool` subtype

## Constraints

- **Transport:** Must use `claude -p` CLI subprocess only — no SDK, no direct API
- **Auth:** Relies on Claude CLI's existing authentication (Pro/Max subscription) — no API key management
- **Compatibility:** Must work with pi's `registerProvider` API and `AssistantMessageEventStream` event contract
- **License:** MIT, with attribution to claude-agent-sdk-pi as the reference implementation
- **Platform:** Node.js (pi extension runtime), must handle subprocess spawning on Windows/macOS/Linux

## Key Decisions

<!-- Decisions that constrain future work. Add throughout project lifecycle. -->

| Decision                                                   | Rationale                                                                             | Outcome    |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------- | ---------- |
| Stateless subprocess (fresh per request)                   | Matches reference project, proven pattern, simpler than persistent sessions           | -- Pending |
| Stream-json control protocol for tool denial               | CLI flags alone cannot achieve "propose but don't execute" (experimentally confirmed) | -- Pending |
| Full parity with reference project's custom tool MCP proxy | Custom pi tools need to be exposed to Claude; MCP proxy is the established pattern    | -- Pending |
| `--input-format stream-json` required                      | Needed for control_response messages to deny/allow tools; cannot drop input formatter | -- Pending |

---

_Last updated: 2026-03-13 after initialization_
