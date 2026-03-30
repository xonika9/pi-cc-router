# Roadmap: pi-claude-cli

## Overview

This roadmap delivers a pi coding agent extension that routes LLM calls through the Claude Code CLI subprocess. The build follows the data flow: first prove the subprocess communication loop (spawn, stream, parse), then layer tool interception and mapping, then complete event coverage (thinking, usage), then expose custom tools via MCP, and finally harden for cross-platform reliability and error handling.

## Phases

**Phase Numbering:**

- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [x] **Phase 1: Core Subprocess Bridge** - Register provider, spawn Claude subprocess, stream text responses back to pi (completed 2026-03-14)
- [x] **Phase 2: Tool Handling** - Intercept tool proposals via control protocol, map names and arguments between Claude and pi (completed 2026-03-14)
- [x] **Phase 3: Extended Thinking and Usage** - Bridge thinking token events and usage metrics with configurable thinking budgets (completed 2026-03-14)
- [x] **Phase 4: Custom Tool MCP Proxy** - Expose pi custom tools to Claude via schema-only stdio MCP server with break-early subprocess control (completed 2026-03-15)
- [x] **Phase 5: Platform Hardening** - Cross-platform subprocess management, error propagation, and image history support (completed 2026-03-15)
- [ ] **Phase 6: Testing and Release Pipeline** - Unit tests, GitHub Actions CI, and automated npm publishing

## Phase Details

### Phase 1: Core Subprocess Bridge

**Goal**: Pi can send a request to the Claude provider and receive streamed text responses end-to-end
**Depends on**: Nothing (first phase)
**Requirements**: PROV-01, PROV-02, PROV-03, PROC-01, STRM-01, STRM-02, HIST-01
**Success Criteria** (what must be TRUE):

1. Extension appears as a selectable provider in pi with all Claude model options listed
2. Sending a simple text prompt through pi produces a streamed text response from Claude
3. Multi-turn conversations work -- pi can send follow-up messages and Claude responds with awareness of prior turns
4. Pi receives proper `text_start`, `text_delta`, `text_end`, and `done` events during streaming
   **Plans:** 3/3 plans complete

Plans:

- [x] 01-01-PLAN.md -- Scaffold project, define shared types, implement prompt builder
- [x] 01-02-PLAN.md -- Implement NDJSON stream parser and event bridge state machine
- [x] 01-03-PLAN.md -- Implement process manager, provider registration, and wire all modules

### Phase 2: Tool Handling

**Goal**: Claude proposes tool calls that pi intercepts, translates, and executes natively -- the "Claude proposes, pi executes" loop works
**Depends on**: Phase 1
**Requirements**: STRM-03, TOOL-01, TOOL-02, TOOL-03, TOOL-04
**Success Criteria** (what must be TRUE):

1. When Claude wants to read a file, pi receives a tool call with the correct pi tool name (`read`) and translated arguments (`path` instead of `file_path`)
2. Claude's built-in tool execution is denied via control protocol -- tools never execute inside the subprocess
3. MCP-prefixed tool calls are allowed through the control protocol without denial
4. Pi receives `toolcall_start`, `toolcall_delta`, and `toolcall_end` events with correct tool names and arguments
   **Plans:** 2/2 plans complete

Plans:

- [x] 02-01-PLAN.md -- Tool mapping module and control protocol handler
- [x] 02-02-PLAN.md -- Wire tool handling into event bridge, provider, and prompt builder

### Phase 3: Extended Thinking and Usage

**Goal**: Pi receives thinking token streams and usage metrics, with configurable thinking effort per model
**Depends on**: Phase 1
**Requirements**: STRM-04, STRM-05, CONF-02
**Success Criteria** (what must be TRUE):

1. When Claude uses extended thinking, pi receives `thinking_start`, `thinking_delta`, and `thinking_end` events with the thinking content
2. After each response, pi receives accurate usage metrics (input tokens, output tokens, cache tokens)
3. Opus models use an elevated thinking effort compared to other models
   **Plans:** 2/2 plans complete

Plans:

- [x] 03-01-PLAN.md -- Thinking effort mapping module, CLI flag wiring, and tests (CONF-02)
- [x] 03-02-PLAN.md -- Verify existing thinking event bridging (STRM-04) and usage metrics (STRM-05)

### Phase 4: Custom Tool MCP Proxy

**Goal**: Pi's custom tools (non-built-in) are available to Claude via MCP, enabling Claude to propose custom tool calls
**Depends on**: Phase 2
**Requirements**: MCP-01, MCP-02, CONF-01
**Success Criteria** (what must be TRUE):

1. Custom pi tools appear in Claude's available tool list via the MCP server
2. When Claude proposes a custom tool call, the `mcp__custom-tools__` prefix is stripped and pi receives the correct tool name
3. Existing `.mcp.json` and `~/.claude.json` MCP configurations load automatically without user intervention
4. At message_stop, subprocess is killed early to prevent CLI from auto-executing tools (break-early pattern)
   **Plans:** 2/2 plans complete

Plans:

- [x] 04-01-PLAN.md -- Schema-only MCP server, config generation, break-early subprocess control, CLI flag changes
- [x] 04-02-PLAN.md -- MCP prefix stripping, prompt builder custom tool replay, extension entry point wiring

### Phase 5: Platform Hardening

**Goal**: Extension works reliably across Windows, macOS, and Linux with proper error handling and edge case coverage
**Depends on**: Phase 1, Phase 2
**Requirements**: PROC-02, PROC-03, PROC-04, HIST-02
**Success Criteria** (what must be TRUE):

1. Extension spawns the Claude CLI correctly on Windows (handling `.cmd` shim resolution via cross-spawn)
2. Subprocess is force-killed after receiving the result event (no orphaned processes)
3. Subprocess crashes, timeouts, and stderr errors surface as error events in pi's stream
4. Conversation history with embedded images replays correctly in the prompt
   **Plans:** 2/2 plans complete

Plans:

- [x] 05-01-PLAN.md -- Process lifecycle hardening: force-kill, inactivity timeout, error surfacing, process registry
- [x] 05-02-PLAN.md -- Image history passthrough with Anthropic format translation

### Phase 6: Testing and Release Pipeline

**Goal**: Extension has comprehensive unit tests, cross-platform CI, and automated npm publishing
**Depends on**: Phase 1, Phase 5
**Requirements**: RLSE-01, RLSE-02, RLSE-03, RLSE-04
**Success Criteria** (what must be TRUE):

1. Unit tests cover NDJSON parsing, event bridging, tool name/argument mapping, control protocol logic, and prompt building with mocked subprocess I/O
2. GitHub Actions CI runs lint, typecheck, and unit tests on Windows, macOS, and Linux runners on every push/PR
3. Tagged releases automatically publish to npm with correct pi-package metadata
4. `package.json` includes `"pi-package"` keyword, `pi.extensions` entry, and correct peer dependencies
   **Plans:** 1/2 plans complete

Plans:

- [x] 06-01-PLAN.md -- Lint/format tooling (ESLint + Prettier), coverage configuration, codebase formatting, test gap filling
- [ ] 06-02-PLAN.md -- GitHub Actions CI and publish workflows, package metadata verification

## Progress

**Execution Order:**
Phases execute in numeric order: 1 -> 2 -> 3 -> 4 -> 5 -> 6
Note: Phase 3 depends only on Phase 1, not Phase 2, so it could theoretically execute in parallel with Phase 2 if needed.

| Phase                           | Plans Complete | Status      | Completed  |
| ------------------------------- | -------------- | ----------- | ---------- |
| 1. Core Subprocess Bridge       | 3/3            | Complete    | 2026-03-14 |
| 2. Tool Handling                | 2/2            | Complete    | 2026-03-14 |
| 3. Extended Thinking and Usage  | 0/2            | Complete    | 2026-03-14 |
| 4. Custom Tool MCP Proxy        | 2/2            | Complete    | 2026-03-15 |
| 5. Platform Hardening           | 2/2            | Complete    | 2026-03-15 |
| 6. Testing and Release Pipeline | 1/2            | In Progress | -          |
