# Phase 3: Extended Thinking and Usage - Research

**Researched:** 2026-03-14
**Domain:** Claude CLI effort/thinking configuration, pi SimpleStreamOptions wiring, stream event verification
**Confidence:** HIGH

## Summary

Phase 3 wires pi's `SimpleStreamOptions` (reasoning level and custom thinking budgets) through to the Claude CLI subprocess, and verifies that existing thinking event bridging (STRM-04) and usage metrics (STRM-05) are correct. The thinking event bridge and usage tracking are already fully implemented in `event-bridge.ts` with passing tests. The primary new work is CONF-02: a `thinking-config.ts` module that maps pi's `ThinkingLevel` to CLI `--effort` values, with elevated defaults for Opus models, plus wiring that budget/effort through `provider.ts` and `process-manager.ts`.

**CRITICAL FINDING:** The Claude CLI does NOT have a `--thinking-budget` flag. The CONTEXT.md references `--thinking-budget <tokens>`, but the actual CLI flag is `--effort <level>` with values `low`, `medium`, `high`, `max` (Opus 4.6 only). The CONTEXT.md's token-based budget approach from the reference project (which used the Agent SDK's `maxThinkingTokens` parameter) does not translate directly to the CLI subprocess approach. The planner must adapt the design to use `--effort` instead. However, the token budget tables from CONTEXT.md can still serve as the *internal mapping logic* -- they just map to effort levels rather than raw token counts on the CLI.

**Primary recommendation:** Map pi's `ThinkingLevel` (minimal/low/medium/high/xhigh) to CLI `--effort` values (low/medium/high/max), with Opus models getting elevated mapping (e.g., pi's "medium" maps to CLI "high" for Opus). The token budget tables from CONTEXT.md become the conceptual rationale for the level mapping, not literal values passed to the CLI.

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions
- Match reference project's two-table approach: standard models and Opus models
- Standard defaults: minimal=2048, low=8192, medium=16384, high=31999
- Opus defaults: minimal=2048, low=8192, medium=31999, high=63999, xhigh=63999
- Opus gets shifted budgets -- "medium" uses standard "high", "high" uses 64K
- Non-Opus models: xhigh silently downgrades to high
- Use `model.id.includes('opus')` for Opus detection -- forward-compatible
- Pass computed budget via CLI flag on the subprocess
- When no thinking level is provided (reasoning undefined), omit the flag entirely
- Phase 3 owns all SimpleStreamOptions wiring into streamViaCli/spawnClaude
- Read `reasoning` and `thinkingBudgets` from the options pi passes to streamSimple
- Custom thinkingBudgets from pi override the default tables (per-level overrides)
- No extension-level configuration surface -- pi's options are the single source of truth
- Provider (streamViaCli) computes the budget number using thinking-config module
- spawnClaude receives just the budget number: `{ thinkingBudget?: number }` in options
- process-manager appends flag only when budget is defined
- Clean separation: config module maps levels to numbers, process-manager maps numbers to CLI flags
- New `src/thinking-config.ts` with budget tables, `mapThinkingTokens()`, and Opus detection
- Exported function signature: `mapThinkingTokens(reasoning?, modelId?, thinkingBudgets?) -> number | undefined`
- Current usage tracking (input/output/cache tokens from message_start and message_delta) is sufficient
- No separate thinking token tracking -- thinking tokens are included in output_tokens
- Use pi's `calculateCost()` as-is
- STRM-04 (thinking blocks) and STRM-05 (usage metrics) are already in event-bridge.ts -- verify, don't rewrite
- Verify both Opus (elevated) and non-Opus (standard) budget paths work correctly

### Claude's Discretion
- Exact error handling for invalid thinking budget values
- Whether to log a warning when xhigh falls back to high on non-Opus models
- Internal module organization within thinking-config.ts

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope

</user_constraints>

<phase_requirements>

## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| STRM-04 | Extension handles extended thinking blocks, bridging `thinking` content block types to pi's `thinking_start/delta/end` events | Already implemented in event-bridge.ts (lines 154-168, 219-268, 287-292). Tests exist at event-bridge.test.ts lines 653-723. Verification only -- no new code needed. |
| STRM-05 | Extension tracks and reports usage metrics (input tokens, output tokens, cache tokens) from `message_start` and `message_delta` events | Already implemented in event-bridge.ts (lines 124-135 for message_start, 327-339 for message_delta). Verification only -- no new code needed. |
| CONF-02 | Extension supports configurable thinking budget per model, with special elevated limits for Opus models | NEW IMPLEMENTATION required. Create thinking-config.ts module, wire SimpleStreamOptions through provider to process-manager, pass `--effort` flag to CLI. |

</phase_requirements>

## CRITICAL: CLI Flag Correction

### The Problem

The CONTEXT.md specifies passing `--thinking-budget <tokens>` to the CLI subprocess. This flag does **not exist** on the Claude CLI (verified against Claude CLI v2.1.76 help output and official docs at https://code.claude.com/docs/en/cli-reference).

### What Actually Exists

The Claude CLI has `--effort <level>` with values:
- `low` -- Minimal thinking, fastest
- `medium` -- Balanced
- `high` -- Deep reasoning (default)
- `max` -- Maximum capability, no token constraints (Opus 4.6 ONLY; other models return error)

**Source:** Claude CLI `--help` output, Claude Code docs (https://code.claude.com/docs/en/model-config#adjust-effort-level), Anthropic API docs (https://platform.claude.com/docs/en/build-with-claude/effort)

### Why the CONTEXT.md Says `--thinking-budget`

The reference project (`claude-agent-sdk-pi`) uses the Claude Agent SDK, which has a `maxThinkingTokens` option (now deprecated in favor of `thinking: { type: "adaptive" }` with `effort`). The CONTEXT.md's token-budget approach was designed for the SDK path. Since pi-claude-cli uses the CLI subprocess directly, the mechanism must be `--effort`.

### Recommended Adaptation

The two-table approach from CONTEXT.md remains valid as the *conceptual design*, but the output changes:

**Instead of:** `mapThinkingTokens()` returning a token number passed as `--thinking-budget <number>`
**Use:** `mapThinkingEffort()` returning an effort string passed as `--effort <level>`

The mapping from pi's `ThinkingLevel` to CLI effort becomes:

**Standard models:**

| Pi ThinkingLevel | CLI --effort | Rationale |
|------------------|-------------|-----------|
| `minimal` | `low` | Minimal thinking |
| `low` | `low` | Matches low |
| `medium` | `medium` | Balanced |
| `high` | `high` | Deep reasoning |
| `xhigh` | `high` | Downgrade (non-Opus) |

**Opus models:**

| Pi ThinkingLevel | CLI --effort | Rationale |
|------------------|-------------|-----------|
| `minimal` | `low` | Minimal thinking |
| `low` | `low` | Matches low |
| `medium` | `high` | Shifted up (Opus elevated) |
| `high` | `max` | Shifted up (Opus gets max) |
| `xhigh` | `max` | Maximum capability |

This preserves the CONTEXT.md's core intent: Opus models get elevated reasoning compared to standard models at the same pi thinking level.

### ThinkingBudgets Override Consideration

Pi's `SimpleStreamOptions.thinkingBudgets` is typed as `ThinkingBudgets` which maps levels to token counts (`{ minimal?: number, low?: number, medium?: number, high?: number }`). Since the CLI does not accept token counts, custom `thinkingBudgets` from pi cannot be honored as exact token values. Options:

1. **Ignore thinkingBudgets entirely** -- simplest, since CLI only accepts effort levels
2. **Use thinkingBudgets as threshold hints** -- if custom budget for a level is above/below certain thresholds, shift the effort level up/down
3. **Log a warning** when thinkingBudgets are provided, explaining they cannot be mapped to CLI effort levels

**Recommendation:** Option 1 (ignore) with a console.warn when custom thinkingBudgets are provided, noting that the CLI subprocess approach only supports effort levels. This is honest and avoids false precision.

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| cross-spawn | ^7.0.6 | Subprocess spawning | Already in project, Windows .cmd shim handling |
| @mariozechner/pi-ai | ^0.52.0 | Types: SimpleStreamOptions, ThinkingLevel, ThinkingBudgets, Usage, calculateCost | pi provider API contract |
| @mariozechner/pi-coding-agent | ^0.52.0 | ExtensionAPI, registerProvider | pi extension registration |

### Supporting
No new dependencies needed. This phase only creates a new TypeScript module and modifies existing files.

## Architecture Patterns

### Recommended Module Structure
```
src/
  thinking-config.ts    # NEW: effort mapping tables + mapThinkingEffort()
  process-manager.ts    # MODIFY: accept effort option, append --effort flag
  provider.ts           # MODIFY: accept SimpleStreamOptions, compute effort, pass to spawnClaude
  event-bridge.ts       # VERIFY ONLY: thinking blocks and usage already implemented
  types.ts              # NO CHANGES needed
index.ts                # MODIFY: pass SimpleStreamOptions through from pi's streamSimple
```

### Pattern 1: Effort Mapping Module (thinking-config.ts)
**What:** Pure function module that maps pi's ThinkingLevel to CLI effort strings
**When to use:** Called by provider.ts during streamViaCli orchestration

```typescript
// src/thinking-config.ts
import type { ThinkingLevel, ThinkingBudgets } from "@mariozechner/pi-ai";

// CLI effort values accepted by --effort flag
export type CliEffortLevel = "low" | "medium" | "high" | "max";

// Standard model mapping: pi ThinkingLevel -> CLI effort
const STANDARD_EFFORT_MAP: Record<ThinkingLevel, CliEffortLevel> = {
  minimal: "low",
  low: "low",
  medium: "medium",
  high: "high",
  xhigh: "high",  // non-Opus: silently downgrade
};

// Opus model mapping: shifted up for elevated reasoning
const OPUS_EFFORT_MAP: Record<ThinkingLevel, CliEffortLevel> = {
  minimal: "low",
  low: "low",
  medium: "high",   // shifted: standard high
  high: "max",      // shifted: maximum capability
  xhigh: "max",     // Opus gets max
};

/**
 * Detect whether a model ID refers to an Opus model.
 * Uses includes('opus') for forward-compatibility with future Opus versions.
 */
export function isOpusModel(modelId: string): boolean {
  return modelId.includes("opus");
}

/**
 * Map pi's ThinkingLevel to a CLI effort string.
 *
 * @param reasoning - Pi's thinking level (undefined = omit flag)
 * @param modelId - Model ID for Opus detection
 * @param thinkingBudgets - Custom budgets (logged as unsupported, not applied)
 * @returns CLI effort level string, or undefined if flag should be omitted
 */
export function mapThinkingEffort(
  reasoning?: ThinkingLevel,
  modelId?: string,
  thinkingBudgets?: ThinkingBudgets,
): CliEffortLevel | undefined {
  if (reasoning === undefined) {
    return undefined; // omit --effort flag entirely
  }

  if (thinkingBudgets && Object.keys(thinkingBudgets).length > 0) {
    console.warn(
      "[pi-claude-cli] Custom thinkingBudgets are not supported with CLI subprocess. " +
      "The CLI uses --effort levels instead of token budgets. Budgets will be ignored."
    );
  }

  const isOpus = modelId ? isOpusModel(modelId) : false;
  const map = isOpus ? OPUS_EFFORT_MAP : STANDARD_EFFORT_MAP;
  return map[reasoning];
}
```

### Pattern 2: Options Wiring Flow
**What:** SimpleStreamOptions flow from pi -> index.ts -> provider.ts -> process-manager.ts
**When to use:** Every streamSimple call

The flow is:
1. `index.ts`: pi calls `streamSimple(model, context, options)` where options is `SimpleStreamOptions`
2. `provider.ts`: `streamViaCli` receives options, calls `mapThinkingEffort(options.reasoning, model.id, options.thinkingBudgets)`, passes effort to `spawnClaude`
3. `process-manager.ts`: `spawnClaude` receives `{ effort?: CliEffortLevel }`, conditionally appends `--effort` flag to args array

### Pattern 3: Conditional CLI Flag Appending
**What:** Only add `--effort` when a value is computed
**When to use:** In process-manager.ts when building CLI args

```typescript
// In spawnClaude, after existing args:
if (options?.effort) {
  args.push("--effort", options.effort);
}
```

### Anti-Patterns to Avoid
- **Don't pass raw token numbers to CLI:** The CLI does not accept `--thinking-budget`. Using it will cause a spawn error.
- **Don't default to an effort level:** When reasoning is undefined, omit the flag entirely to let the CLI use its own default behavior (high).
- **Don't use `--effort max` on non-Opus models:** The CLI returns an error. The mapping tables prevent this by design (non-Opus xhigh maps to high, not max).

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Opus model detection | Regex or version parsing | `model.id.includes('opus')` | Simple, forward-compatible per CONTEXT.md decision |
| Usage cost calculation | Custom pricing logic | `calculateCost(model, output.usage)` from pi-ai | Already handles model-specific pricing, already wired |
| Thinking event streaming | Custom event emitter | Existing event-bridge.ts thinking_start/delta/end | Already implemented and tested (Phase 1) |

## Common Pitfalls

### Pitfall 1: Using --thinking-budget Instead of --effort
**What goes wrong:** CLI spawn fails with unknown flag error
**Why it happens:** Reference project uses Agent SDK's `maxThinkingTokens`, not CLI flags
**How to avoid:** Use `--effort` flag exclusively. Verify against `claude --help` output.
**Warning signs:** Subprocess stderr contains "unknown option" errors

### Pitfall 2: Using --effort max on Non-Opus Models
**What goes wrong:** CLI returns an error instead of a response
**Why it happens:** `max` effort level is Opus 4.6 only (per official docs)
**How to avoid:** Effort mapping tables ensure non-Opus models never get `max`
**Warning signs:** Error events from subprocess, test failures on non-Opus model paths

### Pitfall 3: Not Passing SimpleStreamOptions Through index.ts
**What goes wrong:** Provider never receives reasoning/thinkingBudgets from pi
**Why it happens:** The current `index.ts` passes `options` to `streamViaCli` but the `StreamOptions` type in provider.ts does not include reasoning fields
**How to avoid:** Update provider.ts to accept `SimpleStreamOptions` (or a subset containing reasoning + thinkingBudgets). Update index.ts to ensure the full options object flows through.
**Warning signs:** Effort flag never appears in spawned CLI args

### Pitfall 4: Breaking Existing Tests When Modifying spawnClaude Signature
**What goes wrong:** Existing tests that call spawnClaude with 2-3 args fail
**Why it happens:** Adding new options parameter or changing signature
**How to avoid:** Keep the new effort option optional in the existing options parameter, or add it to the existing `{ cwd?, signal? }` options object
**Warning signs:** process-manager.test.ts failures

### Pitfall 5: Ignoring the `start` Event Already Emitted by Event Bridge
**What goes wrong:** Duplicating start event logic in the new thinking flow
**Why it happens:** Not realizing the event bridge already handles the start event on first handleEvent
**How to avoid:** STRM-04 and STRM-05 are already complete. This phase only needs to verify them.
**Warning signs:** Double "start" events in test output

## Code Examples

### Existing Thinking Event Handling (Verified)

The following is already implemented in `event-bridge.ts` and handles STRM-04:

```typescript
// Source: src/event-bridge.ts lines 154-168 (thinking block start)
} else if (blockType === "thinking") {
  const block: TrackedContentBlock = {
    type: "thinking",
    text: "",
    index: event.index ?? 0,
  };
  blocks.push(block);
  output.content.push({ type: "thinking" as const, thinking: "", thinkingSignature: "" });
  stream.push({
    type: "thinking_start",
    contentIndex: output.content.length - 1,
    partial: output,
  });
}
```

### Existing Usage Tracking (Verified)

The following is already implemented in `event-bridge.ts` and handles STRM-05:

```typescript
// Source: src/event-bridge.ts lines 124-135 (message_start usage)
function handleMessageStart(event: ClaudeApiEvent): void {
  const usage = event.message?.usage;
  if (usage) {
    output.usage.input = usage.input_tokens ?? 0;
    output.usage.output = usage.output_tokens ?? 0;
    output.usage.cacheRead = usage.cache_read_input_tokens ?? 0;
    output.usage.cacheWrite = usage.cache_creation_input_tokens ?? 0;
    output.usage.totalTokens =
      output.usage.input + output.usage.output + output.usage.cacheRead + output.usage.cacheWrite;
    calculateCost(model, output.usage);
  }
}
```

### Existing Signature Accumulation (Verified)

```typescript
// Source: src/event-bridge.ts lines 259-268 (signature_delta)
} else if (deltaType === "signature_delta" && event.delta!.signature != null) {
  const idx = blocks.findIndex((b) => b.index === event.index);
  if (idx === -1) return;
  const block = blocks[idx];
  if (block.type === "thinking") {
    const contentBlock = output.content[idx] as ThinkingContent;
    contentBlock.thinkingSignature = (contentBlock.thinkingSignature || "") + event.delta!.signature;
  }
}
```

### SpawnClaude Modification Pattern

```typescript
// Source: src/process-manager.ts (to be modified)
// Current signature:
export function spawnClaude(
  modelId: string,
  systemPrompt?: string,
  options?: { cwd?: string; signal?: AbortSignal },
): ChildProcess

// New signature (adds effort):
export function spawnClaude(
  modelId: string,
  systemPrompt?: string,
  options?: { cwd?: string; signal?: AbortSignal; effort?: string },
): ChildProcess
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `thinking: { type: "enabled", budget_tokens: N }` | `thinking: { type: "adaptive" }` + `effort` param | Opus 4.6 / Sonnet 4.6 (2026) | budget_tokens deprecated on latest models |
| `maxThinkingTokens` (Agent SDK) | `thinking` config + `effort` option (Agent SDK) | Agent SDK current | maxThinkingTokens marked deprecated |
| No CLI effort flag | `--effort low/medium/high/max` flag | Claude CLI v2.x | CLI now supports effort control |

**Deprecated/outdated:**
- `maxThinkingTokens` in Agent SDK: deprecated, use `thinking` option instead
- `budget_tokens` in API: deprecated on Opus 4.6 and Sonnet 4.6, use adaptive thinking with effort
- `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING=1`: env var exists to revert to fixed budget (escape hatch)

## Open Questions

1. **Custom thinkingBudgets handling**
   - What we know: Pi sends `ThinkingBudgets` (token counts per level), but CLI only accepts effort levels
   - What's unclear: Whether pi extensions in production actually send custom thinkingBudgets
   - Recommendation: Ignore with console.warn. Can revisit if users report needing granular control.

2. **Whether to keep mapThinkingTokens() function name from CONTEXT.md**
   - What we know: CONTEXT.md specifies `mapThinkingTokens()` returning `number | undefined`
   - What's unclear: Whether the planner should honor the function name from CONTEXT.md or rename to reflect actual behavior
   - Recommendation: Rename to `mapThinkingEffort()` returning `CliEffortLevel | undefined` since the function's actual purpose changed. The CONTEXT.md decision about the function's existence and location is honored; the name/return type adapt to reality.

3. **Whether non-Opus xhigh should warn**
   - What we know: CONTEXT.md says "silently downgrades." Claude's Discretion section says researcher should decide about warning.
   - Recommendation: Log a debug-level console.warn (not error) when xhigh downgrades to high on non-Opus models. Visible enough for debugging but not alarming.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Vitest 3.x |
| Config file | vitest.config.ts |
| Quick run command | `npx vitest run --reporter=verbose` |
| Full suite command | `npx vitest run --reporter=verbose` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| STRM-04 | Thinking blocks bridged to pi events (thinking_start/delta/end) | unit | `npx vitest run tests/event-bridge.test.ts -t "thinking" -x` | Yes -- tests exist at lines 653-723 |
| STRM-05 | Usage metrics tracked from message_start and message_delta | unit | `npx vitest run tests/event-bridge.test.ts -t "usage" -x` | Yes -- tests exist (message_start usage, message_delta usage) |
| CONF-02 (mapping) | mapThinkingEffort maps ThinkingLevel to CLI effort for Opus/non-Opus | unit | `npx vitest run tests/thinking-config.test.ts -x` | No -- Wave 0 |
| CONF-02 (wiring) | Provider passes effort through to spawnClaude | unit | `npx vitest run tests/provider.test.ts -x` | Partial -- needs new test cases |
| CONF-02 (flag) | spawnClaude appends --effort flag when effort is defined | unit | `npx vitest run tests/process-manager.test.ts -x` | Partial -- needs new test cases |
| CONF-02 (omit) | No --effort flag when reasoning is undefined | unit | `npx vitest run tests/process-manager.test.ts -x` | Partial -- existing tests verify no extra flags |

### Sampling Rate
- **Per task commit:** `npx vitest run --reporter=verbose`
- **Per wave merge:** `npx vitest run --reporter=verbose`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `tests/thinking-config.test.ts` -- covers CONF-02 mapping (effort tables, Opus detection, xhigh downgrade, undefined reasoning, custom budgets warning)
- [ ] New test cases in `tests/process-manager.test.ts` -- covers --effort flag appending and omission
- [ ] New test cases in `tests/provider.test.ts` -- covers SimpleStreamOptions flow-through to spawnClaude

## Sources

### Primary (HIGH confidence)
- Claude CLI v2.1.76 `--help` output -- verified all available flags, confirmed `--effort` exists and `--thinking-budget` does not
- https://code.claude.com/docs/en/cli-reference -- complete CLI flag reference
- https://code.claude.com/docs/en/model-config -- effort level documentation
- https://platform.claude.com/docs/en/build-with-claude/effort -- effort parameter API docs
- https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking -- adaptive thinking docs (budget_tokens deprecated on Opus 4.6)
- https://platform.claude.com/docs/en/build-with-claude/extended-thinking -- streaming format for thinking blocks
- `@mariozechner/pi-ai` types.d.ts -- SimpleStreamOptions, ThinkingLevel, ThinkingBudgets type definitions
- `@mariozechner/pi-coding-agent` types.d.ts -- ExtensionAPI, ProviderConfig, registerProvider interface
- https://platform.claude.com/docs/en/agent-sdk/typescript -- Agent SDK ThinkingConfig type, effort option

### Secondary (MEDIUM confidence)
- Existing event-bridge.ts implementation -- verified thinking block handling, usage tracking, signature accumulation
- Existing test suite (150 tests passing) -- verified thinking tests at event-bridge.test.ts

### Tertiary (LOW confidence)
- Reference project structure (https://github.com/prateekmedia/claude-agent-sdk-pi) -- could not access source code directly; repository has flat structure with single index.ts; unable to verify exact mapThinkingTokens implementation

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- using existing project dependencies, no new libraries
- Architecture: HIGH -- all pi types verified from source, CLI flags verified from help output and official docs
- Pitfalls: HIGH -- critical `--thinking-budget` vs `--effort` gap identified and verified against multiple authoritative sources
- STRM-04/STRM-05 verification: HIGH -- code reviewed, tests exist and pass

**Research date:** 2026-03-14
**Valid until:** 2026-04-14 (stable -- CLI flags unlikely to change within 30 days)
