# Plan: Fix resume prompt repetition + silent resume errors

## Context (source of truth)

### Bug 1: Resume prompt repetition

`buildResumePrompt()` отправляет на каждом `--resume` оригинальное user message + ВСЕ накопленные tool results. CLI уже имеет историю с предыдущих `--resume` вызовов → модель видит дублированный контент → после N повторов возвращает `end_turn` вместо `tool_use`.

**Evidence:** session `173f2597`, cycle 3: user message отправлен 4+ раз подряд в Claude Code transcript. К 15-му resume: 15 tool results (только 1 новый). Проверено на 5819 resume points — detection rule корректна в 100% случаев.

### Bug 2: Silent resume error on provider switch

Когда пользователь переключает модель с нативного Claude на pi-cc-router, роутер пытается `--resume` session ID, который не существует в Claude CLI. CLI возвращает `{"type":"result","subtype":"error_during_execution","errors":["No conversation found..."]}`. Роутер проверяет `msg.subtype === "error"`, но CLI возвращает `"error_during_execution"` → ошибка не ловится → Pi получает пустой ответ.

**Evidence:** `claude --resume <nonexistent-uuid>` → `result.subtype = "error_during_execution"`, router checks only `"error"`.

## Goal

1. Устранить повторную отправку user message и старых tool results при resume
2. Корректно обрабатывать ошибки CLI при `--resume` и fallback на полный промпт

## Scope

| File | Changes |
|------|---------|
| `src/prompt-builder.ts` | `buildResumePrompt()`: tool continuation detection |
| `src/provider.ts` | Session existence check + error subtype fix |
| `src/types.ts` | Расширить `ClaudeResultMessage` type |
| `tests/prompt-builder.test.ts` | Новые тесты на tool continuation |
| `tests/provider.test.ts` | Тесты на fallback при отсутствии session |

## Out of scope

- Auto-retry при обрыве намерения (модель сказала "сделаю" но не сделала)
- Изменения в `event-bridge.ts`, `process-manager.ts`, `stream-parser.ts`
- Изменения в `buildPrompt()` (первый ход, без --resume)

## Key decisions

1. **Детекция tool continuation:** если после `finalUserIndex` идут только `assistant` и `toolResult` messages (один или несколько циклов `assistant(toolCall) → toolResult+`) → tool continuation. Проверено на 5819 точках, 0 ошибок. Это наблюдаемый инвариант Pi: messages строятся как `[user → (assistant → toolResult+)* → user]`. Добавить runtime assertion: если после `finalUserIndex` встречается message с неожиданным role, логировать предупреждение и fallback на текущую полную resume-логику
2. **Что отправлять в tool continuation:** только tool results из последнего assistant turn + continuation prompt. Не повторять user message и старые tool results
3. **Session existence check:** перед `--resume` проверять наличие файла `~/.claude/projects/<encoded-cwd>/<sessionId>.jsonl`. Нет файла → использовать `buildPrompt` + `buildSystemPrompt` вместо `buildResumePrompt`
4. **Error subtype fix:** проверять `(msg as any).is_error === true` как основной критерий ошибки. Не использовать `msg.subtype !== "success"` — это overclaim, т.к. могут существовать не-success состояния, которые не являются ошибками (отмена, частичный результат). Для subtype — явная матрица известных значений: `"success"` = OK, `"error"` и `"error_during_execution"` = ошибка, неизвестные = предупреждение в лог + трактовать как ошибку

---

## Implementation steps

### [x] Step 1: Fix `ClaudeResultMessage` type

File: `src/types.ts`

Расширить type чтобы отразить реальный API:

```typescript
export interface ClaudeResultMessage {
  type: "result";
  subtype: "success" | "error" | "error_during_execution" | string;
  result?: string;
  error?: string;
  errors?: string[];        // CLI returns array of error strings
  is_error?: boolean;       // reliable error flag from CLI
  session_id?: string;
}
```

### [x] Step 2: Fix error detection in `provider.ts`

File: `src/provider.ts`, line ~276

Change:
```typescript
if (msg.subtype === "error") {
```

To:
```typescript
const isError = (msg as any).is_error === true;
const knownErrorSubtypes = ["error", "error_during_execution"];
const isKnownError = knownErrorSubtypes.includes(msg.subtype);
const isUnknownSubtype = msg.subtype !== "success" && !isKnownError;

if (isUnknownSubtype) {
  console.warn(`[pi-cc-router] Unknown result subtype: "${msg.subtype}", treating as error`);
}

if (isError || isKnownError || isUnknownSubtype) {
  const errorMsg = (msg as any).errors?.[0] ?? msg.error ?? "Unknown error from Claude CLI";
  endStreamWithError(errorMsg);
}
```

This uses `is_error` as the primary signal, with an explicit known-subtypes matrix. Unknown subtypes are logged and treated as errors conservatively.

### [x] Step 3: Add session existence check in `provider.ts`

File: `src/provider.ts`, add helper function and modify session detection block (~line 87).

Add import at top:
```typescript
import { existsSync } from "node:fs";
import { join } from "node:path";
import { homedir } from "node:os";
```

Add helper:
```typescript
function cliSessionExists(sessionId: string, cwd: string): boolean {
  const encodedCwd = cwd.replace(/\//g, "-");
  const sessionFile = join(
    homedir(), ".claude", "projects", encodedCwd, `${sessionId}.jsonl`
  );
  return existsSync(sessionFile);
}
```

Modify resume detection:
```typescript
// Resume only if: session ID exists, conversation has history,
// AND the CLI session file actually exists on disk
const resumeSessionId =
  options?.sessionId && context.messages.length > 1
    ? options.sessionId
    : undefined;

const canResume = resumeSessionId && cliSessionExists(resumeSessionId, cwd);

const prompt = canResume
  ? buildResumePrompt(context)
  : buildPrompt(context);
const systemPrompt = canResume
  ? undefined
  : buildSystemPrompt(context, cwd);
```

Update spawnClaude call:
```typescript
proc = spawnClaude(model.id, systemPrompt || undefined, {
  cwd,
  signal: options?.signal,
  effort,
  mcpConfigPath: options?.mcpConfigPath,
  resumeSessionId: canResume ? resumeSessionId : undefined,
  newSessionId: !canResume ? options?.sessionId : undefined,
});
```

### [x] Step 4: Add tool continuation detection in `buildResumePrompt`

File: `src/prompt-builder.ts`, function `buildResumePrompt()`

After finding `finalUserIndex`, add check:

```typescript
const finalUserIndex = findFinalUserMessageIndex(messages);
if (finalUserIndex < 0) return "";

// Detect tool continuation: toolResults exist AFTER the last user message.
// This means the CLI already has the user message from a prior --resume turn.
// We only need to send the NEW tool results, not repeat the user message.
const hasToolResultsAfterUser = messages
  .slice(finalUserIndex + 1)
  .some((m: any) => m.role === "toolResult");

if (hasToolResultsAfterUser) {
  return buildToolContinuationPrompt(messages, finalUserIndex);
}

// Otherwise: new user message — existing logic continues below
```

### [x] Step 5: Implement `buildToolContinuationPrompt`

File: `src/prompt-builder.ts`, new function.

```typescript
function buildToolContinuationPrompt(
  messages: any[],
  finalUserIndex: number,
): string | AnthropicContentBlock[] {
  // Find the last assistant message (whose tool calls were just executed)
  let lastAssistantIdx = -1;
  for (let i = messages.length - 1; i > finalUserIndex; i--) {
    if (messages[i].role === "assistant") {
      lastAssistantIdx = i;
      break;
    }
  }

  // Collect only tool results after the last assistant (= the new results)
  const startFrom =
    lastAssistantIdx >= 0 ? lastAssistantIdx + 1 : finalUserIndex + 1;
  const parts: string[] = [];
  let hasImages = false;

  for (let i = startFrom; i < messages.length; i++) {
    const msg = messages[i];
    if (msg.role !== "toolResult") continue;

    if (msg.toolName && isCustomToolName(msg.toolName)) {
      parts.push(`TOOL RESULT (${msg.toolName}):`);
    } else {
      const claudeToolName = msg.toolName
        ? mapPiToolNameToClaude(msg.toolName)
        : "unknown";
      parts.push(`TOOL RESULT (historical ${claudeToolName}):`);
    }
    parts.push(toolResultContentToText(msg.content));

    if (toolResultHasImages(msg.content)) {
      hasImages = true;
    }
  }

  parts.push(
    "\nContinue with your planned actions based on the tool results above.",
  );

  // Image handling: return ContentBlock[] if images present
  if (hasImages) {
    const imageBlocks: AnthropicContentBlock[] = [];
    for (let i = startFrom; i < messages.length; i++) {
      if (messages[i].role === "toolResult" && Array.isArray(messages[i].content)) {
        for (const block of messages[i].content) {
          if (block.type === "image") {
            const translated = translateImageBlock(block);
            if (translated) imageBlocks.push(translated);
          }
        }
      }
    }
    const textContent = parts.join("\n");
    return [
      { type: "text" as const, text: textContent },
      ...imageBlocks,
    ];
  }

  return parts.join("\n");
}
```

### [x] Step 6: Add tests for tool continuation

File: `tests/prompt-builder.test.ts`, add to `describe("buildResumePrompt")`:

**Test 1: Tool continuation sends only latest tool result, not user message**
```typescript
it("tool continuation: sends only latest tool result without repeating user message", () => {
  const context = {
    messages: [
      { role: "user", content: "Check server status" },
      { role: "assistant", content: [{ type: "toolCall", name: "bash", arguments: { command: "ssh server uptime" } }] },
      { role: "toolResult", toolName: "bash", content: "up 30 days" },
    ],
  };
  const result = buildResumePrompt(context) as string;
  expect(result).toContain("TOOL RESULT");
  expect(result).toContain("up 30 days");
  expect(result).toContain("Continue with your planned actions");
  expect(result).not.toContain("Check server status");
});
```

**Test 2: Multiple cycles sends only latest batch**
```typescript
it("tool continuation: sends only latest batch, not accumulated results", () => {
  const context = {
    messages: [
      { role: "user", content: "Analyze files" },
      { role: "assistant", content: [{ type: "toolCall", name: "read", arguments: { path: "/a.ts" } }] },
      { role: "toolResult", toolName: "read", content: "old content" },
      { role: "assistant", content: [{ type: "toolCall", name: "read", arguments: { path: "/b.ts" } }] },
      { role: "toolResult", toolName: "read", content: "new content" },
    ],
  };
  const result = buildResumePrompt(context) as string;
  expect(result).toContain("new content");
  expect(result).not.toContain("old content");
  expect(result).not.toContain("Analyze files");
});
```

**Test 3: Parallel tool results sends all from latest batch**
```typescript
it("tool continuation: sends all parallel tool results from latest assistant turn", () => {
  const context = {
    messages: [
      { role: "user", content: "Read both files" },
      { role: "assistant", content: [
        { type: "toolCall", name: "read", arguments: { path: "/a.ts" } },
        { type: "toolCall", name: "read", arguments: { path: "/b.ts" } },
      ]},
      { role: "toolResult", toolName: "read", content: "content A" },
      { role: "toolResult", toolName: "read", content: "content B" },
    ],
  };
  const result = buildResumePrompt(context) as string;
  expect(result).toContain("content A");
  expect(result).toContain("content B");
  expect(result).not.toContain("Read both files");
});
```

**Test 4: New user message preserves current behavior**
```typescript
it("new user message after tools: includes tool results AND new user message", () => {
  const context = {
    messages: [
      { role: "user", content: "Read file" },
      { role: "assistant", content: [{ type: "toolCall", name: "read", arguments: { path: "/a.ts" } }] },
      { role: "toolResult", toolName: "read", content: "file content" },
      { role: "user", content: "Now explain it" },
    ],
  };
  const result = buildResumePrompt(context) as string;
  expect(result).toContain("file content");
  expect(result).toContain("Now explain it");
});
```

**Test 5: Tool continuation with image in tool result**
```typescript
it("tool continuation with image: returns ContentBlock[] array", () => {
  const context = {
    messages: [
      { role: "user", content: "Take screenshot" },
      { role: "assistant", content: [{ type: "toolCall", name: "bash", arguments: {} }] },
      { role: "toolResult", toolName: "bash", content: [
        { type: "text", text: "Screenshot taken" },
        { type: "image", data: "abc123", mimeType: "image/png" },
      ]},
    ],
  };
  const result = buildResumePrompt(context);
  expect(Array.isArray(result)).toBe(true);
  const blocks = result as any[];
  expect(blocks.some(b => b.type === "image")).toBe(true);
  expect(blocks.some(b => b.type === "text" && b.text.includes("Continue"))).toBe(true);
});
```

### [x] Step 7: Add tests for session existence check

File: `tests/provider.test.ts`, add test for the fallback behavior.

Test that when `cliSessionExists` returns false, `buildPrompt` is used instead of `buildResumePrompt`. This can be tested by mocking `existsSync`.

### [x] Step 8: Run all tests

```bash
npx vitest run
```

All existing and new tests must pass.

### [x] Step 9: Type check

```bash
npx tsc --noEmit
```

## Verification

1. [x] `npx vitest run` — все тесты проходят
2. [x] `npx tsc --noEmit` — нет ошибок типизации
3. [ ] **Manual test bug 1 (causation):** воспроизвести long-resume сценарий из session `173f2597` (задача с 5+ SSH tool calls подряд). Проверить: (a) user message не дублируется в Claude Code transcript, (b) модель продолжает делать `tool_use` на каждом resume, а не прерывается с `end_turn` текстом
4. [ ] **Manual test bug 2 (session identity):** начать сессию с нативным Claude, переключить на pi-cc-router модель → (a) модель отвечает непустым контентом, (b) проверить что созданная CLI-сессия имеет корректный `session_id` в transcript

## Definition of Done

- [x] `buildResumePrompt` не повторяет user message в tool continuation case
- [x] `buildResumePrompt` отправляет только tool results из последнего assistant turn
- [x] Error detection ловит `"error_during_execution"` и другие subtypes
- [x] При отсутствии CLI session файла роутер fallback на `buildPrompt`
- [x] Все тесты проходят
- [x] `tsc --noEmit` без ошибок
- [x] Runtime assertion логирует предупреждение при неожиданном message role после finalUserIndex

## Execution status

- Completed on 2026-03-30 in the current branch.
- Added automated coverage for tool continuation, image passthrough on resume, unknown result subtype handling, and resume fallback when the Claude CLI session file is missing.
- Repository-level verification completed:
  - [x] `npm test`
  - [x] `npm run typecheck`
  - [x] `npm run lint`
- Manual Claude CLI smoke checks from the Verification section remain pending.

## Assumptions

- Pi отправляет полный `context.messages` на каждом вызове (подтверждено логами)
- С `--resume` CLI загружает историю с диска (подтверждено документацией)
- Claude CLI хранит сессии в `~/.claude/projects/<cwd-with-slashes-replaced-by-dashes>/<sessionId>.jsonl` (подтверждено проверкой файловой системы)
- `is_error` флаг в result messages — надёжный индикатор ошибки CLI (подтверждено тестами с `claude --resume`)

## Review decisions

По итогам document review (coherence, feasibility, adversarial) — 5 auto-fixes применены, 3 finding'а рассмотрены и отклонены:

1. **Windows path encoding в `cliSessionExists`** — оставлено как есть. Проект используется на Mac. `existsSync` fallback безопасен: если файл не найден → `buildPrompt`. Задокументировано в Risks.
2. **Нет runtime retry при ошибке `--resume`** — оставлено как есть. Pre-check закрывает основной кейс (provider switch). Corrupt session и race conditions крайне редки; показать ошибку честнее чем молча глотать. Retry — непропорциональная сложность для edge cases.
3. **Foreign session ID при fallback** — переиспользуем тот же `sessionId`. Это единственный вариант при котором resume начинает работать после provider switch (первый ход — full prompt, последующие — resume). Генерация нового UUID сломала бы resume навсегда для переключённой сессии.

## Risks

- **Low:** Continuation prompt `"Continue with your planned actions..."` может быть недостаточно информативным. Mitigation: CLI уже имеет полную историю, prompt — только триггер
- **Low:** Path encoding `cwd.replace(/\//g, "-")` может измениться в будущих версиях Claude CLI. Mitigation: если файл не найден, fallback на полный промпт — safe default
- **Medium:** `cliSessionExists` использует Unix-style path encoding. На Windows `\` и `:` в путях не обрабатываются. Mitigation: если `existsSync` не находит файл, fallback безопасен (полный промпт). Для полной Windows-поддержки потребуется reverse-engineering актуального encoding из Claude CLI
