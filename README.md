# pi-cc-router

Route [pi](https://github.com/mariozechner/pi-coding-agent) LLM calls through the [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code). Use your Claude Pro/Max subscription as the backend — no API key required.

## Installation

```
pi install npm:pi-cc-router
```

Select a Claude model via `/model`. All Claude models appear under the `pi-cc-router` provider.

## Requirements

- Claude Code CLI installed and authenticated (`claude` on PATH)
- Claude Pro or Max subscription

## How it works

Each request spawns a `claude -p` subprocess using the stream-json protocol. Claude proposes tool calls — pi executes them natively. Custom pi tools are exposed to Claude via a schema-only MCP server.

- Streams text, thinking, and tool call tokens in real-time
- Maps tool names and arguments bidirectionally between Claude and pi
- Configurable thinking effort with elevated budgets for Opus models
- Cross-platform: Windows, macOS, Linux

## License

MIT
