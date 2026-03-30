# Phase 3: Extended Thinking and Usage - Context

**Gathered:** 2026-03-14
**Status:** Ready for planning

<domain>
## Phase Boundary

Bridge thinking token events and usage metrics with configurable thinking budgets per model. Accept pi's `SimpleStreamOptions` (reasoning level + custom budgets) and pass computed thinking budget to the Claude CLI via `--thinking-budget` flag. Opus models get elevated budgets. STRM-04 and STRM-05 are already implemented in the event bridge — this phase verifies them and implements CONF-02.

</domain>

<decisions>
## Implementation Decisions

### Thinking Budget Tables

- Match reference project's two-table approach: standard models and Opus models
- Standard defaults: minimal=2048, low=8192, medium=16384, high=31999
- Opus defaults: minimal=2048, low=8192, medium=31999, high=63999, xhigh=63999
- Opus gets shifted budgets — "medium" uses standard "high", "high" uses 64K
- Non-Opus models: xhigh silently downgrades to high

### Opus Model Detection

- Use `model.id.includes('opus')` — catches opus-4-6, opus-4.6, future opus-5, etc.
- Forward-compatible: any Opus model gets elevated budgets, not just 4.6

### CLI Flag

- Pass computed budget via `--thinking-budget <tokens>` flag on the subprocess
- When no thinking level is provided (reasoning undefined), omit the flag entirely — let CLI use its own default behavior
- Researcher should verify exact flag syntax and accepted values

### SimpleStreamOptions Wiring

- Phase 3 owns all SimpleStreamOptions wiring into streamViaCli/spawnClaude
- Read `reasoning` and `thinkingBudgets` from the options pi passes to streamSimple
- Custom thinkingBudgets from pi override the default tables (per-level overrides)
- No extension-level configuration surface — pi's options are the single source of truth

### API Shape

- Provider (streamViaCli) computes the budget number using thinking-config module
- spawnClaude receives just the budget number: `{ thinkingBudget?: number }` in options
- process-manager appends `--thinking-budget` flag only when budget is defined
- Clean separation: config module maps levels → numbers, process-manager maps numbers → CLI flags

### Configuration Module

- New `src/thinking-config.ts` with budget tables, `mapThinkingTokens()`, and Opus detection
- Exported function signature: `mapThinkingTokens(reasoning?: ThinkingLevel, modelId?: string, thinkingBudgets?: ThinkingBudgets) → number | undefined`

### Usage Metrics

- Current tracking (input/output/cache tokens from message_start and message_delta) is sufficient
- No separate thinking token tracking — thinking tokens are included in output_tokens
- Use pi's `calculateCost()` as-is — pi's library handles model-specific pricing
- Researcher should verify usage field handling against reference project to catch any gaps

### Existing Code Validation

- STRM-04 (thinking blocks) and STRM-05 (usage metrics) are already in event-bridge.ts
- Verify existing implementation against reference project during research — fix gaps, don't rewrite
- Verify both Opus (elevated) and non-Opus (standard) budget paths work correctly

### Claude's Discretion

- Exact error handling for invalid thinking budget values
- Whether to log a warning when xhigh falls back to high on non-Opus models
- Internal module organization within thinking-config.ts

</decisions>

<specifics>
## Specific Ideas

- Reference project (`claude-agent-sdk-pi`) has a proven `mapThinkingTokens()` function — replicate its logic with the Opus detection broadened from `opus-4-6`/`opus-4.6` to any `opus` model
- The reference project's comment: "opus-4-6 gets shifted budgets so 'high' uses the budget that xhigh would normally use"
- Pi's `supportsXhigh()` doesn't recognize non-Anthropic-API provider types, so xhigh may arrive from pi but should be handled gracefully

</specifics>

<code_context>

## Existing Code Insights

### Reusable Assets

- `event-bridge.ts`: Already handles thinking_start/delta/end events and signature accumulation — no changes needed for STRM-04
- `event-bridge.ts`: Already tracks usage from message_start and message_delta with calculateCost — no changes needed for STRM-05
- `types.ts`: ClaudeApiEvent already supports thinking_delta and signature_delta delta types

### Established Patterns

- Process manager builds CLI args as an array and passes to cross-spawn — budget flag follows same pattern
- Provider orchestrates options → process manager → subprocess — budget wiring follows same flow
- Event bridge uses parallel block tracking with content index correlation

### Integration Points

- `provider.ts` streamViaCli: Needs to accept SimpleStreamOptions, compute budget, pass to spawnClaude
- `process-manager.ts` spawnClaude: Needs new thinkingBudget option, conditionally appends --thinking-budget flag
- `index.ts`: Provider registration may need to pass options through from pi's streamSimple call

</code_context>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

_Phase: 03-extended-thinking-and-usage_
_Context gathered: 2026-03-14_
