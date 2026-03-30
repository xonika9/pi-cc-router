---
status: resolved
trigger: "internal-tool-timeout: When the model uses internal tools (Agent, Skill, ToolSearch, Task*), pi-claude-cli silently filters them and the inactivity timer kills the subprocess before the CLI finishes executing them internally."
created: 2026-03-21T00:00:00Z
updated: 2026-03-21T00:00:00Z
---

## Current Focus

hypothesis: CONFIRMED as root cause -- two interacting problems cause the timeout
test: Code trace through provider.ts + event-bridge.ts + tool-mapping.ts
expecting: N/A -- root cause confirmed
next_action: Implement fix in provider.ts

## Symptoms

expected: Model's text response displays after internal tool (Agent/Skill) completes in the CLI subprocess
actual: Spinner shows "working", then drops to prompt with no output after 180s
errors: No explicit error -- subprocess killed by inactivity timeout
reproduction: Run any GSD command that dispatches sub-agents (e.g. /gsd:verify-work)
started: Has never worked -- internal tools were always filtered, but the timeout interaction makes it fatal

## Eliminated

## Evidence

- timestamp: 2026-03-21T00:01:00Z
  checked: provider.ts line 226-230 -- inactivity timer reset logic
  found: resetInactivityTimer() is called on EVERY raw stdout line (line 230), BEFORE parseLine(). This means the timer resets on any NDJSON line, including internal tool events.
  implication: The timer IS being reset during the initial streaming of the internal tool_use block events. The gap happens AFTER the CLI finishes streaming the model response and starts EXECUTING the internal tool.

- timestamp: 2026-03-21T00:02:00Z
  checked: provider.ts line 239-249 -- break-early tracking
  found: sawBuiltInOrCustomTool only set to true for isPiKnownClaudeTool() tools. Internal tools (Agent, Task, etc.) do NOT set this flag.
  implication: At message_stop (line 252), break-early does NOT fire for internal-only tool calls. The subprocess stays alive.

- timestamp: 2026-03-21T00:03:00Z
  checked: The subprocess lifecycle after message_stop with internal tools
  found: After message_stop with sawBuiltInOrCustomTool=false, the readline loop keeps running. The CLI subprocess begins executing the internal tool (Agent/Task/etc.). During this execution, the CLI produces NO NDJSON stream events on stdout (it's between model turns). If this execution takes > 180s, the inactivity timer fires and kills the process.
  implication: This is the root cause. The CLI is alive and working, but silently executing an internal tool with no stdout output.

- timestamp: 2026-03-21T00:04:00Z
  checked: event-bridge.ts line 181-188 -- internal tool filtering
  found: handleContentBlockStart returns early (no block tracked, no event pushed) for tools where isPiKnownClaudeTool() returns false. Subsequent deltas and stop for these tools are silently dropped because blocks.findIndex returns -1.
  implication: The event bridge correctly filters internal tools, but this means the provider has no visibility into whether an internal tool is "in progress" -- it can't use this to extend the timeout.

- timestamp: 2026-03-21T00:05:00Z
  checked: provider.ts line 281-306 -- post-readline done event logic
  found: Lines 287-293 already handle the case where stopReason is "toolUse" but no pi-known tools exist: it overrides to "stop". This means if the CLI eventually completes the internal tool execution and produces a result, the done event would use effectiveReason="stop".
  implication: The multi-turn output plumbing should work IF the process survives long enough. The only blocker is the inactivity timeout killing the process mid-execution.

- timestamp: 2026-03-21T00:06:00Z
  checked: What events the CLI produces during internal tool execution
  found: Claude CLI in stream-json mode emits NDJSON for each model turn's streaming events. Between turns (during tool execution), it may emit system messages, but crucially for internal tools like Agent that spawn sub-agents, the execution can take many minutes with no stdout.
  implication: Need to detect when an internal tool is in-flight and either extend or disable the timeout.

## Resolution

root_cause: When the model uses only internal tools (Agent, Skill, Task, ToolSearch), the break-early logic does NOT fire (because sawBuiltInOrCustomTool is false). The CLI subprocess stays alive to execute the tool internally. During this internal execution, no stdout is produced for extended periods. The 180s inactivity timer fires and kills the subprocess, terminating the entire request with no output.
fix: |
In provider.ts:

1. Added INTERNAL_TOOL_TIMEOUT_MS constant (600_000ms = 10 minutes)
2. Added sawInternalTool flag to track when internal tools (Agent, Task, etc.) are seen
3. Added activeTimeoutMs variable (starts at 180s, switches to 600s for internal tools)
4. At content_block_start for non-pi-known tools, set sawInternalTool = true
5. At message_stop when only internal tools were seen (no break-early), extend timeout to 600s and reset timer
6. At message_start (new model turn), restore activeTimeoutMs to normal 180s
7. resetInactivityTimer() uses activeTimeoutMs instead of hardcoded INACTIVITY_TIMEOUT_MS
   verification: |

- All 298 tests pass (295 existing + 3 new)
- TypeScript compiles cleanly
- New tests cover: extended timeout for internal tools, multi-turn scenario with agent then text, break-early still works for pi-known tools
  files_changed:
- src/provider.ts
- tests/provider.test.ts
