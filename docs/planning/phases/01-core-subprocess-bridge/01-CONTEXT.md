# Phase 1: Core Subprocess Bridge - Context

**Gathered:** 2026-03-13
**Status:** Ready for planning

<domain>
## Phase Boundary

Register as a custom pi provider, spawn `claude -p` with stream-json protocol, parse NDJSON output, bridge text streaming events to pi's `AssistantMessageEventStream`, and replay conversation history for multi-turn awareness. Tool handling, thinking, usage metrics, MCP proxy, and platform hardening are separate phases.

</domain>

<decisions>
## Implementation Decisions

### Provider Identity

- Provider ID: `pi-claude-cli` (follows reference project convention of using the package name)
- Models: derived dynamically from `getModels("anthropic")` — auto-updates when pi adds new models
- Display names: unchanged from Anthropic catalog (no suffix)
- `api` field: `"pi-claude-cli"` (matches provider concept)
- `baseUrl` and `apiKey`: placeholder strings (not used for actual HTTP/auth)

### History Serialization

- Flatten entire conversation history into a single text prompt with USER:/ASSISTANT:/TOOL RESULT: labels (matches reference project's proven approach)
- Send as a single user message to the subprocess

### System Prompt

- Pass pi's system context through to Claude via `--append-system-prompt` or equivalent
- Also append AGENTS.md content and skill definitions from pi config (matching reference project behavior)
- Sanitize path references (`.pi` → `.claude`) for Claude Code compatibility

### CLI Flags

- `--no-session-persistence` — stateless model, no reason to save sessions to disk
- `--model` — pass selected pi model ID directly (matching reference project's model selection approach)
- Match reference project defaults for other flags where applicable to the CLI (e.g., `--permission-mode dontAsk`)
- Required flags: `--input-format stream-json`, `--output-format stream-json`, `--verbose`

### Startup Validation

- Check for Claude CLI presence on PATH at provider registration time
- If missing: show clear error with install instructions and fail gracefully (don't register a broken provider)
- Check Claude CLI authentication status at startup (run `claude auth status` or similar)
- If not authenticated: warn user with login instructions
- Don't silently register a provider that will fail on first request

### Claude's Discretion

- NDJSON parsing error handling details (malformed lines, partial JSON)
- Exact subprocess spawn options (stdio configuration, environment variables)
- Stream event buffering strategy
- Internal error message wording

</decisions>

<specifics>
## Specific Ideas

- Reference project (`claude-agent-sdk-pi`) is the primary architecture reference — follow its patterns for provider registration, model derivation, history serialization, and system prompt handling
- The stream-json wire protocol requires `--verbose` for granular streaming events (`stream_event` wrappers with raw API events)
- `--include-partial-messages` flag needed for real-time token streaming (content_block_delta events)

</specifics>

<code_context>

## Existing Code Insights

### Reusable Assets

- None — greenfield project, no existing code

### Established Patterns

- None — first phase establishes all patterns

### Integration Points

- Pi's `registerProvider()` API is the sole integration point
- `streamSimple` function signature from pi's `ExtensionAPI` defines the contract
- `AssistantMessageEventStream` from `@mariozechner/pi-ai` defines the output event format

</code_context>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

_Phase: 01-core-subprocess-bridge_
_Context gathered: 2026-03-13_
