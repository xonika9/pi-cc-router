# Feature Landscape

**Domain:** Custom LLM provider extension for pi coding agent, routing through Claude CLI subprocess with tool call bridging
**Researched:** 2026-03-13

## Table Stakes

Features users expect. Missing = product feels incomplete or broken.

| Feature                                        | Why Expected                                                                                                                                                  | Complexity | Notes                                                                                                                                                                                                                     |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Subprocess spawning and lifecycle management   | Without it, nothing works. Must spawn `claude -p` with correct flags and clean up on exit/error                                                               | Medium     | Cross-platform (Windows/macOS/Linux). Windows has known issues with console window visibility (`windowsHide: true` needed) and PATH resolution with nvm4w. Node.js `child_process.spawn` is the mechanism                 |
| Stream-json wire protocol handling             | The entire communication channel. Must parse NDJSON from stdout, write NDJSON to stdin                                                                        | Medium     | `--output-format stream-json --input-format stream-json --verbose --include-partial-messages` flags required for granular token streaming                                                                                 |
| Real-time token streaming                      | Pi users expect to see tokens appear as they're generated, not batch responses. All other pi providers stream                                                 | High       | Must bridge Claude API stream events (`content_block_start/delta/stop`, `message_start/delta/stop`) to pi's `AssistantMessageEventStream` events (`text_start/delta/end`, `done`, `error`)                                |
| Tool call bridging (propose-but-don't-execute) | Core architectural requirement. Claude proposes tool calls; pi executes them natively in TUI. Without this, you get duplicate execution or no tool use at all | High       | Uses `control_request`/`control_response` protocol. Deny built-in tools (`behavior: "deny"`), allow MCP tools (`behavior: "allow"` for `mcp__` prefix). Experimentally confirmed that CLI flags alone cannot achieve this |
| Bidirectional tool name mapping                | Claude uses different tool names than pi (Read vs read, Glob vs find). Without mapping, tool calls fail silently or error out                                 | Medium     | 6 built-in mappings: Read<->read, Write<->write, Edit<->edit, Bash<->bash, Grep<->grep, Glob<->find                                                                                                                       |
| Tool argument translation                      | Even with correct tool names, argument shapes differ between Claude and pi                                                                                    | Medium     | Key mappings: `file_path`<->`path`, `old_string`<->`oldText`, `new_string`<->`newText`, `command`<->`command` (same), `pattern`<->`pattern` (same). Must handle both directions                                           |
| Provider registration with model list          | Extension must register via `pi.registerProvider()` with all available Claude models and their properties                                                     | Low        | Models from `getModels("anthropic")`. Each needs: id, name, reasoning flag, input types, cost metrics, contextWindow, maxTokens                                                                                           |
| Conversation history replay                    | Stateless subprocess requires full history to be sent each request. Without it, Claude has no context of prior conversation                                   | High       | Flatten pi's structured conversation history into a single prompt with USER/ASSISTANT/TOOL RESULT labels. This is the most token-expensive aspect of the stateless model                                                  |
| Error handling and graceful failure            | Subprocess crashes, timeouts, malformed JSON, unexpected exits. Users expect clear error messages, not silent failures                                        | Medium     | Must handle: process exit codes, stderr parsing, NDJSON parse failures, timeout scenarios, and emit proper `error` events to pi                                                                                           |
| Authentication passthrough                     | Users expect to use their existing Claude CLI login (Pro/Max subscription). No API key setup, no extra auth flows                                             | Low        | Relies entirely on `claude` CLI's existing auth. Just needs to verify the CLI is installed and authenticated                                                                                                              |

## Differentiators

Features that set this extension apart from alternatives. Not strictly expected, but valued.

| Feature                               | Value Proposition                                                                                                                     | Complexity | Notes                                                                                                                                                                                                                                                                                             |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Custom tool MCP proxy                 | Expose pi's custom/extension-registered tools to Claude via an MCP server, so Claude can call tools that only exist in pi's ecosystem | High       | Reference project uses `createSdkMcpServer()` from the Agent SDK. This project needs an alternative: likely a stdio MCP server spawned alongside the CLI, registered via `--mcp-config`. Claude references these as `mcp__custom-tools__<toolName>`, which must be remapped back to pi tool names |
| Existing MCP config passthrough       | Allow Claude to use MCP servers already configured in `~/.claude.json` and `.mcp.json` without extra setup                            | Low        | `strictMcpConfig` defaults to `false` (opposite of reference project's `true` default). This is a deliberate differentiator -- users who already have MCP servers configured for Claude Code get them automatically                                                                               |
| Extended thinking/reasoning support   | Stream Claude's thinking tokens to pi's `thinking_start/delta/end` events, supporting reasoning models                                | Medium     | Pi supports thinking levels ("off" through "xhigh"). Models must declare `reasoning: true`. Claude API sends thinking content blocks that need separate event bridging. Provides visibility into Claude's reasoning process                                                                       |
| System prompt customization           | Allow appending pi's AGENTS.md and skills to Claude's system prompt, or letting Claude load its own from `.claude/`                   | Low        | Reference project has `appendSystemPrompt` config (default: true). The `--system-prompt` flag controls what the subprocess sees. Trade-off: appending pi context gives better tool guidance, but increases token usage                                                                            |
| Image input support                   | Pass images from pi conversation to Claude via the subprocess                                                                         | Medium     | Claude supports image inputs. Models must declare `input: ["text", "image"]`. Requires serializing image data into the flattened prompt format. Valuable for UI/design workflows                                                                                                                  |
| Configurable MCP strictness           | Toggle between strict (only extension-specified MCP servers) and permissive (load from user/project configs)                          | Low        | `strictMcpConfig` setting. Strict mode prevents tool schema dumps from auto-loaded MCP servers (which can be noisy). Permissive mode is more user-friendly for those with existing setups                                                                                                         |
| Token/cost tracking passthrough       | Surface Claude's token usage and cost data back through pi's tracking system                                                          | Medium     | Claude API responses include usage data. Pi tracks costs per provider. Pro/Max subscription users still benefit from seeing token consumption even without per-token billing, for context window awareness                                                                                        |
| Subprocess process cleanup on Windows | Properly terminate the Claude CLI subprocess tree on Windows where `SIGTERM` doesn't work                                             | Medium     | Windows subprocess termination is notoriously tricky. Need `taskkill /t /f /pid` or tree-kill npm package. Without this, orphaned processes accumulate (documented: 280 zombie processes in one case)                                                                                             |

## Anti-Features

Features to explicitly NOT build. These are tempting but wrong for this project.

| Anti-Feature                                   | Why Avoid                                                                                                                                                                                                                                                                              | What to Do Instead                                                                                                                                                        |
| ---------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Persistent subprocess sessions                 | Adds significant complexity, unproven with stream-json protocol, breaks the stateless model the reference project validated. Risk of context desync between pi and the subprocess. The "50K tokens per turn" problem is real but the cure (persistent sessions) introduces harder bugs | Stay stateless. Each pi request spawns a fresh `claude -p` subprocess with full history replayed. Accept the token overhead as a trade-off for simplicity and reliability |
| Direct Anthropic API calls                     | Defeats the entire purpose of the extension (using Pro/Max subscription without API key). Also duplicates what pi already has with its built-in Anthropic provider                                                                                                                     | Always route through `claude` CLI subprocess. Never import `@anthropic-ai/sdk` directly                                                                                   |
| Claude Agent SDK dependency                    | The reference project uses this. But it adds a heavy dependency, ties to Anthropic's SDK release cycle, and the SDK is essentially a wrapper around spawning the CLI anyway                                                                                                            | Use direct CLI subprocess with `child_process.spawn`. Fewer dependencies, more control, same underlying mechanism                                                         |
| Custom authentication flows                    | Users already authenticate via `claude` CLI. Building auth management creates maintenance burden and security surface area for zero user value                                                                                                                                         | Check that `claude` CLI is installed and authenticated. If not, tell the user to run `claude login`. Done                                                                 |
| Model fine-tuning or custom model registration | Claude doesn't support fine-tuned models through the CLI. Adding custom model endpoints would require API access, which contradicts the project premise                                                                                                                                | Only expose models available through `getModels("anthropic")`. If a model isn't available through the CLI, it's not available                                             |
| Automatic retry with provider fallback         | This extension is specifically for Claude via CLI. There's no fallback provider. Retrying a crashed subprocess is fine; falling back to a different LLM is out of scope                                                                                                                | Implement basic subprocess restart on crash (up to N retries). Let pi's own provider fallback handle cross-provider scenarios                                             |
| Prompt caching management                      | Prompt caching is handled by Claude's backend automatically. The CLI subprocess can't control cache breakpoints. Trying to optimize caching at the extension level adds complexity with no lever to pull                                                                               | Trust Claude's automatic prompt caching. Structure the flattened prompt consistently (system prompt first, then history) so caching works naturally across turns          |
| Tool execution within the subprocess           | The entire architecture is "Claude proposes, pi executes." Letting Claude also execute tools creates duplicate actions, permission conflicts, and breaks pi's TUI rendering of tool results                                                                                            | Always deny built-in tool execution via control protocol. Only allow `mcp__` prefixed tools (MCP servers that Claude manages directly)                                    |

## Feature Dependencies

```
Provider Registration
  |-> Model List (must know available models to register)
  |-> streamSimple handler (the core streaming function)
        |-> Subprocess Spawning (need a process to communicate with)
        |     |-> Cross-platform process management (Windows/macOS/Linux)
        |     |-> Process cleanup/termination
        |-> Stream-json Protocol Handling (parse/write NDJSON)
        |     |-> Real-time Token Streaming (bridge API events to pi events)
        |     |-> Extended Thinking Streaming (bridge thinking blocks)
        |-> Conversation History Replay (build the prompt)
        |     |-> Image Input Support (serialize images into prompt)
        |-> Tool Call Bridging (control protocol)
        |     |-> Tool Name Mapping (bidirectional)
        |     |-> Tool Argument Translation (bidirectional)
        |     |-> Custom Tool MCP Proxy (expose pi tools to Claude)
        |           |-> MCP Config Passthrough (load existing MCP servers)
        |-> Error Handling (process crashes, parse failures)
        |-> Token/Cost Tracking (surface usage data)
```

**Critical path:** Provider Registration -> Subprocess Spawning -> Stream-json Protocol -> Token Streaming -> Tool Call Bridging -> Tool Name/Argument Mapping

**Secondary path (can be deferred):** Custom Tool MCP Proxy, Extended Thinking, Image Input, System Prompt Customization

## MVP Recommendation

Prioritize (in build order):

1. **Subprocess spawning with lifecycle management** - Foundation. Nothing works without a running `claude -p` process that can be cleanly started and stopped
2. **Stream-json protocol parsing (NDJSON read/write)** - Communication layer. Must parse stdout and write to stdin reliably
3. **Real-time token streaming bridge** - User-facing value. Map Claude API stream events to pi's `AssistantMessageEventStream` format (`text_start/delta/end`, `done`, `error`)
4. **Conversation history replay** - Context. Flatten pi's message history into the prompt so Claude knows what happened before
5. **Tool call bridging via control protocol** - Core value. Deny built-in tools, surface tool proposals to pi for native execution
6. **Bidirectional tool name and argument mapping** - Correctness. Without this, tool proposals from Claude can't be understood by pi and vice versa
7. **Provider registration with full model list** - Integration point. Wire everything up via `pi.registerProvider()`
8. **Error handling and graceful degradation** - Polish. Handle subprocess crashes, malformed output, timeouts

Defer:

- **Custom tool MCP proxy**: Complex, requires spawning/configuring an additional MCP server. Can be added after basic tool bridging works. Users lose access to pi's custom tools in Claude's reasoning, but built-in tools cover 90%+ of coding workflows
- **Extended thinking support**: Nice to have, but text streaming is the priority. Thinking events can be added incrementally
- **Image input**: Uncommon in CLI-based coding workflows. Add when text-based flows are solid
- **System prompt customization**: Default behavior (Claude loads its own prompt) works fine initially. Customization can come later
- **Token/cost tracking**: Informational, not functional. Add after core streaming is reliable

## Complexity Assessment

| Feature                     | Estimated Effort | Risk Level | Notes                                                                                                                                                                           |
| --------------------------- | ---------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Subprocess spawning         | 2-3 days         | Medium     | Windows cross-platform is the main risk. Known issues with `windowsHide`, PATH resolution, and process tree cleanup                                                             |
| Stream-json protocol        | 2-3 days         | Low        | NDJSON is well-understood. Main challenge is handling partial lines and backpressure                                                                                            |
| Token streaming bridge      | 3-4 days         | High       | Event mapping between two different streaming formats. Edge cases with interleaved content blocks, tool use blocks, and thinking blocks. Must handle partial messages correctly |
| Conversation history replay | 1-2 days         | Medium     | Flattening is straightforward but prompt formatting must be precise. Token overhead grows linearly with conversation length                                                     |
| Tool call bridging          | 3-4 days         | High       | Control protocol timing is critical. Must respond to `control_request` synchronously on stdin. Race conditions possible between stdout parsing and stdin writing                |
| Tool name/argument mapping  | 1-2 days         | Low        | Static mappings, well-documented. Main risk is undocumented arguments or new tools added to Claude                                                                              |
| Provider registration       | 1 day            | Low        | Straightforward API call. Model list is mostly static                                                                                                                           |
| Error handling              | 2-3 days         | Medium     | Many failure modes to handle. Subprocess exit, malformed JSON, timeout, stderr output                                                                                           |
| Custom tool MCP proxy       | 4-5 days         | High       | Requires spawning an MCP server, configuring it via `--mcp-config`, handling tool schema registration, and remapping `mcp__custom-tools__` prefixed names back to pi tool names |
| Extended thinking           | 1-2 days         | Low        | Once token streaming works, thinking blocks are just another content block type to map                                                                                          |
| Image input                 | 2-3 days         | Medium     | Serialization format for images in the flattened prompt needs investigation                                                                                                     |

## Sources

- [pi-mono custom provider docs](https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/custom-provider.md) - Provider registration API, model properties, streamSimple signature
- [pi-mono extensions docs](https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/extensions.md) - Extension lifecycle, hooks, capabilities
- [claude-agent-sdk-pi](https://github.com/prateekmedia/claude-agent-sdk-pi) - Reference implementation: tool mapping, MCP proxy, configuration options
- [Claude Code CLI reference](https://code.claude.com/docs/en/cli-reference) - CLI flags, stream-json format, control protocol
- [Claude Code headless mode docs](https://code.claude.com/docs/en/headless) - Programmatic CLI usage, streaming flags
- [Claude Agent SDK TypeScript - Windows windowsHide issue](https://github.com/anthropics/claude-agent-sdk-typescript/issues/103) - Windows subprocess visibility problem
- [Claude Code subprocess token overhead analysis](https://dev.to/jungjaehoon/why-claude-code-subagents-waste-50k-tokens-per-turn-and-how-to-fix-it-41ma) - Token cost of stateless subprocess model, system prompt re-injection
- [Claude Code Node.js spawning issue](https://github.com/anthropics/claude-code/issues/771) - Cross-platform child_process challenges
- [pi-mono extension types](https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/src/core/extensions/types.ts) - TypeScript types for extension API
- [Claude API streaming docs](https://platform.claude.com/docs/en/build-with-claude/streaming) - Content block delta event format
