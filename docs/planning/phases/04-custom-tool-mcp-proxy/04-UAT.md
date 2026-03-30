---
status: complete
phase: 04-custom-tool-mcp-proxy
source: [04-01-SUMMARY.md, 04-02-SUMMARY.md]
started: 2026-03-15T04:00:00Z
updated: 2026-03-15T04:15:00Z
---

## Current Test

<!-- OVERWRITE each test - shows where we are -->

[testing complete]

## Tests

### 1. Custom Tool Visible to Claude

expected: Load pi with both pi-claude-cli AND test-tool-extension.ts. Send "What's the weather in London?" Claude should propose the weather tool — pi shows a toolcall event with name "weather" (prefix stripped).
result: pass

### 2. Built-in Tools Handled by Pi

expected: Send a query that triggers a built-in tool (e.g., "Read the file package.json"). Pi should execute the Read tool itself (via its agent loop), NOT Claude CLI. You should see pi's standard tool execution output, not Claude returning file contents directly in its response text.
result: pass

### 3. Basic Query Without Tools

expected: Send a simple query that doesn't require tools (e.g., "What is 2+2?"). Claude should respond normally with text. No tool calls, no errors. This confirms the extension still works for basic queries even with MCP changes.
result: pass

### 4. Extension Loads Without Custom Tools

expected: Load pi with ONLY pi-claude-cli (no test-tool-extension). Send any query. The extension should work normally — no errors about MCP setup failure. The warn-don't-block pattern means MCP failures are silently handled.
result: [pending]

### 7. Display: Custom tool calls render as text instead of blue/green boxes

expected: Custom tool calls should display as blue boxes (proposed) and green boxes (executed), same as built-in tools.
result: issue
reported: "Custom tool calls show as text ([Used weather tool with args...]) instead of blue/green boxes. Race condition: done event pushed async after readline closes, creating timing gap between toolcall events and done event."
severity: minor

### 5. User MCP Tools Still Work

expected: If you have MCP servers configured in your .mcp.json or ~/.claude.json, send a query that would use one of those tools. Claude should execute it via the MCP server normally (not blocked by break-early). If you don't have user MCPs configured, skip this test.
result: pass

### 6. Custom Tool Result in Conversation History

expected: After test 1 (weather tool), send a follow-up message referencing the previous tool result (e.g., "What about Paris?"). Claude should have context from the previous weather result and propose the tool again. This confirms custom tool results are replayed in conversation history with the correct MCP prefix.
result: pass

## Summary

total: 6
passed: 5
issues: 1
pending: 1
skipped: 0

## Gaps

- truth: "Claude discovers custom tool via MCP and proposes using it"
  status: failed
  reason: "User reported: Claude loops calling ToolSearch trying to fetch MCP tool schema, gets 'Tool ToolSearch not found' error, infinite loop requiring ctrl-c to kill"
  severity: blocker
  test: 1
  root_cause: "Event bridge emits ToolSearch toolcall to pi. Pi doesn't know ToolSearch (it's a Claude Code internal tool). Pi returns error. Claude retries. Infinite loop. The event bridge needs to filter out internal Claude Code tools (ToolSearch, Task, TaskOutput, etc.) and only emit toolcall events for pi-known tools (6 built-ins + custom tools)."
  artifacts:
  - path: "src/event-bridge.ts"
    issue: "Emits toolcall events for ALL tool_use blocks including internal Claude Code tools"
    missing:
  - "Filter internal Claude Code tools from event bridge output — only emit toolcall for pi-known tools"
    debug_session: ""
