# Requirements: pi-claude-cli

**Defined:** 2026-03-13
**Core Value:** Enable pi users to leverage their Claude Pro/Max subscription as the LLM backend via the official CLI

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Provider Registration

- [x] **PROV-01**: Extension registers as a custom pi provider via `pi.registerProvider()` with a unique provider ID
- [x] **PROV-02**: Provider exposes all current Claude models derived from `getModels("anthropic")` with correct context windows, max tokens, and cost info
- [x] **PROV-03**: Provider implements `streamSimple` handler as the core entry point for LLM requests

### Subprocess Management

- [x] **PROC-01**: Extension spawns `claude -p --input-format stream-json --output-format stream-json --verbose` as a fresh subprocess per pi request
- [x] **PROC-02**: Extension uses `cross-spawn` for subprocess creation to handle Windows `.cmd` shim resolution
- [x] **PROC-03**: Extension force-kills the subprocess after receiving the `result` event (known CLI hang bug in stream-json mode)
- [x] **PROC-04**: Extension propagates subprocess crashes, timeouts, and stderr errors to pi as error events on the `AssistantMessageEventStream`

### Stream Protocol

- [x] **STRM-01**: Extension parses NDJSON output from the Claude subprocess line-by-line via `readline`, validating each line as JSON before processing
- [x] **STRM-02**: Extension bridges Claude API stream events (`content_block_start/delta/stop`, `message_start/delta/stop`) to pi's `AssistantMessageEventStream` events (`text_start/delta/end`, `toolcall_start/delta/end`, `thinking_start/delta/end`, `done`/`error`)
- [x] **STRM-03**: Extension sends `control_response` messages on subprocess stdin to deny or allow tool execution in response to `control_request` messages on stdout
- [x] **STRM-04**: Extension handles extended thinking blocks, bridging `thinking` content block types to pi's `thinking_start/delta/end` events
- [x] **STRM-05**: Extension tracks and reports usage metrics (input tokens, output tokens, cache tokens) from `message_start` and `message_delta` events

### Tool Handling

- [x] **TOOL-01**: Extension denies all built-in tool execution requests via control protocol (`behavior: "deny"`) -- Claude proposes, pi executes
- [x] **TOOL-02**: Extension allows `mcp__` prefixed tool execution requests via control protocol (`behavior: "allow"`) -- Claude executes MCP tool calls itself
- [x] **TOOL-03**: Extension maps tool names bidirectionally between Claude built-in names and pi equivalents (Read<->read, Write<->write, Edit<->edit, Bash<->bash, Grep<->grep, Glob<->find)
- [x] **TOOL-04**: Extension translates tool arguments between Claude and pi formats (e.g., `file_path`<->`path`, `old_string`<->`oldText`, `head_limit`<->`limit`)

### Conversation History

- [x] **HIST-01**: Extension builds a flattened prompt from the full pi conversation history, including USER, ASSISTANT, and TOOL RESULT blocks with proper role labeling
- [x] **HIST-02**: Extension includes base64-encoded images in the replayed prompt when present in pi conversation messages

### Custom Tool MCP Proxy

- [x] **MCP-01**: Extension exposes custom pi tools (non-built-in) to Claude via a stdio MCP server, registered with the subprocess via `--mcp-config`
- [x] **MCP-02**: Extension strips the `mcp__custom-tools__` prefix from tool call names when mapping MCP tool proposals back to pi tool names

### Configuration

- [x] **CONF-01**: Extension defaults `strictMcpConfig` to `false`, allowing existing `.mcp.json` and `~/.claude.json` MCP server configurations to load automatically
- [x] **CONF-02**: Extension supports configurable thinking budget per model, with special elevated limits for Opus models

### Testing and Release

- [x] **RLSE-01**: Project includes unit tests with mocked subprocess I/O covering NDJSON parsing, event bridging, tool name/argument mapping, control protocol logic, and prompt building
- [x] **RLSE-02**: GitHub Actions CI pipeline runs lint, typecheck, and unit tests across Windows, macOS, and Linux runners
- [x] **RLSE-03**: GitHub Actions automates npm publish on tagged releases with `"pi-package"` keyword and correct `pi.extensions` entry
- [x] **RLSE-04**: `package.json` structured for pi compatibility — `"pi-package"` keyword, `pi.extensions` pointing to entry file, `@mariozechner/pi-ai` and `@mariozechner/pi-coding-agent` as peer dependencies

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Performance

- **PERF-01**: Persistent subprocess sessions to avoid ~12s startup overhead per request and O(n^2) token growth from replaying full conversation history each turn
- **PERF-02**: Conversation history truncation strategy for long sessions approaching context limits
- **PERF-03**: Process pooling for concurrent pi requests

### Enhanced Features

- **FEAT-01**: Model-specific system prompt customization
- **FEAT-02**: Streaming cost estimation before request completion

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature                          | Reason                                                                     |
| -------------------------------- | -------------------------------------------------------------------------- |
| Direct Anthropic API calls       | The entire point is avoiding API key requirements                          |
| Claude Agent SDK dependency      | Replaced by direct CLI subprocess; SDK is what the reference project uses  |
| Custom authentication flows      | Relies on existing `claude` CLI auth (Pro/Max subscription)                |
| Persistent subprocess sessions   | Adds complexity, unproven with stream-json protocol, defer to v2           |
| Tool execution within subprocess | Breaks core "Claude proposes, pi executes" architecture (except MCP tools) |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase   | Status   |
| ----------- | ------- | -------- |
| PROV-01     | Phase 1 | Complete |
| PROV-02     | Phase 1 | Complete |
| PROV-03     | Phase 1 | Complete |
| PROC-01     | Phase 1 | Complete |
| PROC-02     | Phase 5 | Complete |
| PROC-03     | Phase 5 | Complete |
| PROC-04     | Phase 5 | Complete |
| STRM-01     | Phase 1 | Complete |
| STRM-02     | Phase 1 | Complete |
| STRM-03     | Phase 2 | Complete |
| STRM-04     | Phase 3 | Complete |
| STRM-05     | Phase 3 | Complete |
| TOOL-01     | Phase 2 | Complete |
| TOOL-02     | Phase 2 | Complete |
| TOOL-03     | Phase 2 | Complete |
| TOOL-04     | Phase 2 | Complete |
| HIST-01     | Phase 1 | Complete |
| HIST-02     | Phase 5 | Complete |
| MCP-01      | Phase 4 | Complete |
| MCP-02      | Phase 4 | Complete |
| CONF-01     | Phase 4 | Complete |
| CONF-02     | Phase 3 | Complete |
| RLSE-01     | Phase 6 | Complete |
| RLSE-02     | Phase 6 | Complete |
| RLSE-03     | Phase 6 | Complete |
| RLSE-04     | Phase 6 | Complete |

**Coverage:**

- v1 requirements: 26 total
- Mapped to phases: 26
- Unmapped: 0

---

_Requirements defined: 2026-03-13_
_Last updated: 2026-03-13 after roadmap creation_
