---
title: "fix: Deny Claude internal tools to prevent CLI hanging"
type: fix
status: active
date: 2026-03-30
origin: .planning/debug/cli-hanging-internal-tools.md
---

# fix: Deny Claude internal tools to prevent CLI hanging

## Overview

CLI зависает на ~180с когда Claude использует internal tools (Agent, ToolSearch, Task и т.д.). Pi не умеет их выполнять, break-early не срабатывает, CLI выполняет их внутри себя без stdout → таймер бездействия убивает процесс. Фикс: запретить internal tools через control handler + defense-in-depth на таймере.

## Problem Frame

Пошаговый trace из origin документа:

1. Claude стримит `tool_use: Agent` → `isPiKnownClaudeTool("Agent")` = false → `sawBuiltInOrCustomTool` остаётся false
2. На `message_stop` break-early НЕ срабатывает → подпроцесс продолжает работу
3. CLI шлёт `control_request` для Agent → `control-handler.ts` отвечает "allow"
4. CLI выполняет Agent внутри себя → минуты тишины → таймер убивает процесс

Решение (Путь A из origin): на шаге 3 ответить "deny" вместо "allow". Claude получит отказ и переформулирует через прямые инструменты (Read, Grep, Glob), которые pi выполняет сам.

## Requirements Trace

- R1. Internal tools (Agent, ToolSearch, Task, TaskOutput, TodoWrite, NotebookEdit, ExitPlanMode, AskUserQuestion, Skill, WebFetch, WebSearch, RemoteTrigger, SendMessage) должны получать `deny` на `control_request`
- R2. Custom MCP tools (`mcp__custom-tools__*`) по-прежнему получают `deny` (существующее поведение)
- R3. User MCP tools и built-in tools по-прежнему получают `allow` (существующее поведение)
- R4. Defense-in-depth: таймер бездействия НЕ должен сбрасываться суб-агентными stream-событиями (`parent_tool_use_id != null`). Только `stream_event` сообщения имеют это поле; остальные типы (`control_request`, `result`, `system`) — штатные top-level сообщения, которые должны сбрасывать таймер
- R5. Все существующие тесты проходят (с обновлениями для изменённого поведения)

## Scope Boundaries

- НЕ реализуем upstream PR #18 (показ прогресса суб-агентов)
- НЕ добавляем расширенный таймаут для internal tools
- НЕ меняем event-bridge.ts — фильтрация internal tools в event bridge остаётся как есть
- НЕ меняем промпт или system prompt

## Context & Research

### Relevant Code and Patterns

- `src/control-handler.ts` — уже реализует deny для `mcp__custom-tools__*` через `isCustomTool` + тернарник в `response`. Нужно расширить условие на internal tools
- `src/tool-mapping.ts` — содержит `isPiKnownClaudeTool()` и `CUSTOM_TOOLS_MCP_PREFIX`. Новый хелпер `isClaudeInternalTool()` следует тому же паттерну (Set + функция-предикат)
- `src/provider.ts:228-230` — `resetInactivityTimer()` вызывается для каждой строки stdout ДО проверки `isTopLevel`. Нужно перенести внутрь `isTopLevel` блока
- `tests/control-handler.test.ts:96` — тест "allows internal tools like ToolSearch" → нужно инвертировать ожидание

### Institutional Learnings

- `.planning/debug/internal-tool-timeout.md` — предыдущая попытка фикса (увеличить таймаут) была задокументирована как resolved но никогда не закоммичена. Путь A — более радикальное и полное решение

## Key Technical Decisions

- **Deny all currently known internal tools, не выборочно**: Путь C (оставить ToolSearch) отвергнут — ToolSearch не нужен pi, Claude знает инструменты из system prompt. Единый Set проще поддерживать
- **Список internal tools из origin документа + codebase**: Set основан на origin `cli-hanging-internal-tools.md` раздел "Путь A" (12 tools) + `TaskOutput` из существующих тестов `event-bridge.test.ts:960`. Итого 13 tool names
- **Defense-in-depth на таймере**: Даже если deny не сработает (новый tool, баг в CLI), таймер не будет бесконечно сбрасываться суб-агентными событиями

## Open Questions

### Resolved During Planning

- **Нужно ли менять deny message?** Нет — используем тот же `TOOL_EXECUTION_DENIED_MESSAGE` что и для custom tools. Claude получит единообразный отказ

### Deferred to Implementation

- **Может ли Claude застрять в цикле повторных попыток при deny?** Маловероятно — Claude обучен обрабатывать tool denials. Но стоит проверить вручную после реализации

## Implementation Units

- [x] **Unit 1: Добавить `isClaudeInternalTool()` в tool-mapping.ts**

**Goal:** Единая точка определения internal tools Claude Code

**Requirements:** R1

**Dependencies:** Нет

**Files:**

- Modify: `src/tool-mapping.ts`
- Test: `tests/tool-mapping.test.ts`

**Approach:**

- Добавить `const CLAUDE_INTERNAL_TOOLS = new Set([...13 names...])` после существующего `BUILT_IN_PI_NAMES`
- Добавить экспортируемую функцию `isClaudeInternalTool(toolName: string): boolean`
- Паттерн: аналогичен существующему `isCustomToolName()` — Set + предикат

**Patterns to follow:**

- `isCustomToolName()` в том же файле — тот же стиль: Set для O(1) lookup, чистая функция-предикат

**Test scenarios:**

- Happy path: `isClaudeInternalTool("Agent")` → true, `isClaudeInternalTool("ToolSearch")` → true, `isClaudeInternalTool("Task")` → true
- Happy path: проверить все 13 имён из Set
- Edge case: `isClaudeInternalTool("Read")` → false (built-in pi tool)
- Edge case: `isClaudeInternalTool("mcp__custom-tools__foo")` → false (custom tool)
- Edge case: `isClaudeInternalTool("SomeUnknownTool")` → false (неизвестный tool)

**Verification:**

- `npm test -- tests/tool-mapping.test.ts` проходит
- Функция экспортируется и доступна для импорта

---

- [x] **Unit 2: Запретить internal tools в control-handler.ts**

**Goal:** CLI получает `deny` при запросе на выполнение internal tools

**Requirements:** R1, R2, R3

**Dependencies:** Unit 1

**Files:**

- Modify: `src/control-handler.ts`
- Test: `tests/control-handler.test.ts`

**Approach:**

- Импортировать `isClaudeInternalTool` из `./tool-mapping.js`
- Заменить `isCustomTool` в тернарнике на `shouldDeny = isCustomTool || isClaudeInternalTool(toolName)`
- Вернуть `!shouldDeny` вместо `!isCustomTool`

**Patterns to follow:**

- Существующая логика deny для custom tools — тот же response format, тот же `TOOL_EXECUTION_DENIED_MESSAGE`

**Test scenarios:**

- Happy path: `Agent` → denied (behavior: "deny", message: TOOL_EXECUTION_DENIED_MESSAGE), return false
- Happy path: `ToolSearch` → denied, return false
- Happy path: `Task` → denied, return false
- Happy path: `mcp__custom-tools__weather` → по-прежнему denied (R2 — регрессии нет)
- Happy path: `Read` → по-прежнему allowed (R3)
- Happy path: `mcp__database__query` → по-прежнему allowed (R3)
- Edge case: `SomeUnknownTool` → по-прежнему allowed (не internal, не custom)
- **Обновить существующий тест** строка 96: "allows internal tools like ToolSearch" → инвертировать на "denies internal tools like ToolSearch"

**Verification:**

- `npm test -- tests/control-handler.test.ts` проходит
- Все существующие тесты для custom tools и user MCP tools не сломаны

---

- [x] **Unit 3: Defense-in-depth — таймер только для top-level событий в provider.ts**

**Goal:** Таймер бездействия не сбрасывается суб-агентными событиями

**Requirements:** R4

**Dependencies:** Нет (независим от Units 1-2, но реализуется после для чистоты)

**Files:**

- Modify: `src/provider.ts`

**Approach:**

- В обработчике `rl.on("line", ...)` перенести `resetInactivityTimer()` из общего блока (где сейчас вызывается для каждой строки stdout) внутрь проверки `isTopLevel` для stream_event, оставив вызов для других типов сообщений (control_request, result, system)
- Конкретно: убрать `resetInactivityTimer()` со строки ~230 (до parseLine), и вызывать его: (a) для не-stream_event сообщений после parseLine, (b) для top-level stream_event внутри `if (isTopLevel)` блока

**Patterns to follow:**

- Существующая проверка `isTopLevel` в том же файле

**Test scenarios:**

- Happy path: top-level stream_event по-прежнему сбрасывает таймер
- Edge case: sub-agent stream_event (`parent_tool_use_id != null`) НЕ сбрасывает таймер и процесс завершается по inactivity timeout
- Integration: `tests/provider.test.ts` покрывает top-level и sub-agent flow без изменений public API

**Verification:**

- `npm run typecheck` проходит (`package.json` не содержит `build`, поэтому compile-equivalent проверка в этом репозитории — `tsc --noEmit`)
- `npm test -- tests/provider.test.ts` проходит
- Логика: суб-агентные события (`parent_tool_use_id != null`) НЕ сбрасывают таймер

## System-Wide Impact

- **Interaction graph:** control-handler.ts — единственная точка, где CLI получает allow/deny. Изменение затрагивает только internal tools, не влияет на custom tools или user MCP tools
- **Error propagation:** При deny Claude получает error result для tool call → модель переформулирует через доступные инструменты. Штатное поведение Claude API
- **Unchanged invariants:** break-early логика в provider.ts не меняется. Event bridge фильтрация не меняется. Prompt builder не меняется

## Risks & Dependencies

| Risk                                                             | Mitigation                                                                            |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| Claude может повторно пытаться вызвать denied tool               | Маловероятно — модели обучены обрабатывать denials. Проверить при ручном тестировании |
| Новый internal tool в будущей версии Claude CLI не попадёт в Set | Defense-in-depth (Unit 3) защищает от таймаута. Set легко расширить                   |

## Sources & References

- **Origin document:** [.planning/debug/cli-hanging-internal-tools.md](.planning/debug/cli-hanging-internal-tools.md)
- Related upstream: [rchern/pi-claude-cli#12](https://github.com/rchern/pi-claude-cli/issues/12), [PR #18](https://github.com/rchern/pi-claude-cli/pull/18)
- Related code: `src/tool-mapping.ts`, `src/control-handler.ts`, `src/provider.ts`
