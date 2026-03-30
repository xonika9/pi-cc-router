---
status: open
trigger: "CLI hangs when Claude uses internal tools (Agent, ToolSearch, etc.) — user sees text response then nothing happens for ~180s until timeout kills the process"
created: 2026-03-29
upstream_issue: https://github.com/rchern/pi-claude-cli/issues/12
upstream_pr: https://github.com/rchern/pi-claude-cli/pull/18
related_issue: https://github.com/rchern/pi-claude-cli/issues/8
related_pr: https://github.com/rchern/pi-claude-cli/pull/11
---

# CLI Hanging When Claude Uses Internal Tools

## Симптомы

Пользователь пишет запрос в pi. Claude отвечает текстом ("Хорошо, сделаю пару параллельных поисков") — и после этого ничего не происходит. Выглядит как обрыв связи. Через ~180 секунд процесс убивается таймером бездействия.

Воспроизведение: любой промпт, который побуждает Claude использовать инструмент Agent (например, запрос на масштабный поиск по кодовой базе).

## Корневая причина

Пошаговый trace того, что происходит:

### Нормальный flow (pi-known tools: Read, Grep, etc.)

1. Pi отправляет запрос в Claude CLI подпроцесс
2. Claude стримит ответ: текст + `tool_use: Read`
3. `provider.ts` видит Read → `isPiKnownClaudeTool("Read")` = true → `sawBuiltInOrCustomTool` = true
4. На `message_stop` срабатывает **break-early**: процесс убивается, pi получает tool call
5. Pi сам выполняет Read, показывает результат пользователю
6. Всё быстро, пользователь видит прогресс

### Сломанный flow (internal tools: Agent, ToolSearch, Task, etc.)

1. Pi отправляет запрос в Claude CLI подпроцесс
2. Claude стримит ответ: текст "Сделаю параллельные поиски" + `tool_use: Agent`
3. `provider.ts` видит Agent → `isPiKnownClaudeTool("Agent")` = **false** → `sawBuiltInOrCustomTool` остаётся **false**
4. `event-bridge.ts` фильтрует Agent (строка 186: `if (!isPiKnownClaudeTool(claudeName)) return`) — блок не трекается, события не генерируются
5. На `message_stop` break-early **НЕ срабатывает** (sawBuiltInOrCustomTool = false) → подпроцесс продолжает работу
6. CLI шлёт `control_request` для Agent → `control-handler.ts` отвечает "allow" (Agent — не custom MCP tool)
7. CLI начинает выполнять Agent **внутри себя** — это может занять минуты
8. Во время выполнения CLI может генерировать суб-агентные события (с `parent_tool_use_id`), но `provider.ts` их фильтрует: `const isTopLevel = !(msg as any).parent_tool_use_id` — они не отправляются в event bridge
9. Суб-агентные события **сбрасывают таймер бездействия** (строка 230: `resetInactivityTimer()` вызывается для каждой строки до фильтрации). Но если суб-агентных событий нет (CLI молчит между model turns) — таймер НЕ сбрасывается
10. **Через 180 секунд тишины** → таймер бездействия убивает процесс → стрим обрывается без результата

### Почему pi не может выполнить Agent сам

Pi не знает инструмент "Agent". У pi свой набор: read, write, edit, bash, grep, find. Agent — это внутренний инструмент Claude Code ("запусти ещё один Claude внутри себя"). Pi сам управляет циклом запрос/ответ/инструменты — ему не нужен агент внутри агента. Если Claude хочет три поиска — он должен вернуть три `tool_use: Grep`, pi их выполнит.

## Затронутые файлы

| Файл                          | Роль в проблеме                                        |
| ----------------------------- | ------------------------------------------------------ |
| `src/provider.ts:243-270`     | Break-early логика: не срабатывает для internal tools  |
| `src/provider.ts:226-230`     | Inactivity timer: сбрасывается на каждую строку stdout |
| `src/event-bridge.ts:186-187` | Фильтрация: internal tools пропускаются                |
| `src/control-handler.ts:53`   | Разрешение: internal tools получают "allow"            |
| `src/tool-mapping.ts:55-58`   | `isPiKnownClaudeTool()`: Agent/ToolSearch/Task = false |

## Ранее задокументированная попытка

В `.planning/debug/internal-tool-timeout.md` эта проблема была диагностирована (2026-03-21) и помечена как "resolved". Запланированный фикс:

- Увеличить таймаут до 600с (10 минут) для внутренних инструментов
- Добавить `sawInternalTool` флаг и `activeTimeoutMs` переменную
- Переключать таймаут при обнаружении internal tool

**Но этот фикс так и не был закоммичен** — в текущем коде нет ни одного из описанных изменений. Git log подтверждает: ни одного коммита с "internal", "timeout" или "sawInternal".

## Upstream (rchern/pi-claude-cli)

### Issue #8 (закрыт) — "Agent/Skill Tool Calls Silently Dropped"

https://github.com/rchern/pi-claude-cli/issues/8

Та же проблема. Решён через PR #11.

### PR #11 (merged) — "fix: scope break-early to top-level events only"

https://github.com/rchern/pi-claude-cli/pull/11

Уже в нашем коде (коммит `ada9d3b`). Суть: break-early теперь срабатывает только на top-level события, суб-агентные события (parent_tool_use_id != null) игнорируются. Это позволило CLI выполнять internal tools без преждевременного kill.

### Issue #12 (открыт) — "Show sub-agent progress instead of silent 'Working...'"

https://github.com/rchern/pi-claude-cli/issues/12

Продолжение проблемы. После PR #11 CLI больше не убивается break-early, но пользователь ничего не видит — только "Working..." на долгое время. Предложены три подхода:

1. Event forwarding — передавать суб-агентные события как новый тип
2. Status text emissions — генерировать статусы ("Agent: reading src/...")
3. Custom renderer — рендерить дерево вложенных tool calls

### PR #18 (открыт, на ревью) — "Show sub-agent progress for issue #12"

https://github.com/rchern/pi-claude-cli/pull/18

Upstream-решение: показывать прогресс суб-агентов. Реализация:

- Новый модуль `subagent-progress.ts` — трекает суб-агентные события, формирует статусные сообщения
- `event-bridge.ts` — новый метод `emitEphemeralStatus()` для временных статусных текстовых блоков (помечены `__ephemeral` флагом)
- `provider.ts` — суб-агентные события передаются в progress tracker; `getOutput()` заменён на `getFinalOutput()` (фильтрует ephemeral-блоки)
- Пользователь видит "Agent: searching files..." вместо тишины
- Ephemeral-блоки убираются из финального AssistantMessage

## Два пути решения

### Путь A: Запретить internal tools на уровне control handler

**Суть**: когда CLI спрашивает разрешение на Agent/ToolSearch/Task — отвечать "deny". Claude получит отказ и переформулирует через Grep/Glob/Read, которые pi умеет выполнять.

**Изменения (~15 строк + тесты):**

1. `src/tool-mapping.ts` — добавить Set внутренних инструментов + хелпер `isClaudeInternalTool()`:

   ```typescript
   const CLAUDE_INTERNAL_TOOLS = new Set([
     "Agent",
     "ToolSearch",
     "Task",
     "TodoWrite",
     "NotebookEdit",
     "ExitPlanMode",
     "AskUserQuestion",
     "Skill",
     "WebFetch",
     "WebSearch",
     "RemoteTrigger",
     "SendMessage",
   ]);

   export function isClaudeInternalTool(toolName: string): boolean {
     return CLAUDE_INTERNAL_TOOLS.has(toolName);
   }
   ```

2. `src/control-handler.ts` — запретить internal tools:

   ```typescript
   import { isClaudeInternalTool } from "./tool-mapping.js";
   // ...
   const shouldDeny = isCustomTool || isClaudeInternalTool(toolName);
   ```

   Использовать `shouldDeny` вместо `isCustomTool`.

3. `src/provider.ts` — defense-in-depth: не сбрасывать таймер на не-top-level события:

   ```typescript
   const isTopLevel =
     msg.type !== "stream_event" || !(msg as any).parent_tool_use_id;
   if (isTopLevel) {
     resetInactivityTimer();
   }
   ```

4. Тесты:
   - `tests/control-handler.test.ts` — тест "allows internal tools like ToolSearch" (строка 96) → ожидать denial; добавить тесты для Agent, Task
   - `tests/tool-mapping.test.ts` — тесты для `isClaudeInternalTool()`

**Плюсы:**

- Простая реализация
- Полностью устраняет зависание
- Pi контролирует все операции и показывает реальный прогресс
- Claude умеет делать всё то же через прямые инструменты

**Минусы:**

- Claude теряет Agent (автономные multi-step цепочки внутри одного вызова)
- Потенциально менее эффективен для сложных задач (больше round-trips между pi и Claude)

### Путь B: Показывать прогресс суб-агентов (upstream подход из PR #18)

**Суть**: не запрещать internal tools, а отслеживать суб-агентные события и показывать пользователю временные статусы.

**Изменения (~100+ строк + новый модуль + тесты):**

1. Новый файл `src/subagent-progress.ts` — трекер прогресса
2. `src/event-bridge.ts` — метод `emitEphemeralStatus()`, `getFinalOutput()`, `contentIndex` tracking
3. `src/provider.ts` — интеграция трекера, обработка суб-агентных событий
4. `src/types.ts` — новые типы

**Плюсы:**

- Claude сохраняет Agent-возможности
- Пользователь видит прогресс (хоть и приблизительный)
- Совместимость с upstream

**Минусы:**

- Значительно сложнее
- Pi не контролирует операции — просто показывает лог чужого процесса
- Таймаут всё ещё может быть проблемой в edge cases (если нет суб-агентных событий)
- Агент внутри агента — архитектурно избыточно для pi

### Путь C: Комбинация

Запретить Agent/Task (долгие операции) + оставить ToolSearch (быстрый, нужен CLI для резолва deferred tools). Плюс таймерный фикс как defense-in-depth.

## Рекомендация

**Путь A (запрет)** — лучший для pi. Pi сам является агентом и управляет циклом инструментов. Agent внутри CLI дублирует эту функциональность. Запрет заставляет Claude использовать прямые инструменты, которые pi умеет выполнять и показывать в реальном времени.

## Верификация после фикса

1. `npm run build` — компиляция без ошибок
2. `npm test` — все тесты проходят (существующие обновлены + новые)
3. Ручной тест: промпт, побуждающий Claude использовать Agent → Claude должен переформулировать через Grep/Glob/Read
