# Разбор дизайна харнеса Claude Code

В статье [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) Anthropic чётко заявляет, что надёжность comes from the harness, not the model, and that agents need constraints «outside the model». Claude Code — это продуктовый пример этой идеи; Anthropic itself explicitly classifies it as an **agentic harness**. This is not marketing language — Claude Code may be the most thoroughly examined public harness today: its source code is open, community research reports are detailed, and it turns nearly every core mechanism in the course lectures — layered memory, context compaction, permissions, hooks, subagents, and session persistence — into a complete product implementation.

В этой статье мы используем фреймворк пяти подсистем курса для разбора Claude Code, focusing on how it implements foundational harness concepts such as «context management», «preventing premature declarations of completion», and «deterministic constraints».

## В одном предложении

В основе Claude Code лежит простой while-цикл: вызов модели, выполнение инструмента, наблюдение за результатом и повторный вызов модели. Но **большая часть кода находится не в этом цикле; она в системах, окружающих цикл** — система разрешений, конвейер сжатия контекста, механизмы расширений, оркестрация субагентов и хранилище сессий. Это суть харнеса: цикл — это скелет, но всё вокруг скелета определяет надёжность.

## Подсистема инструкций: многоуровневая система памяти

Система памяти Claude Code — это его самый прямой вклад в теорию харнесов, соответствующий лекциям курса о «репозитории как системе записи» и «преемственности контекста между сессиями». Официальная документация [How Claude remembers your project](https://code.claude.com/docs/en/memory) чётко states that every session starts with a fresh context window and carries knowledge across sessions through two mechanisms: CLAUDE.md files (instructions you write) and auto memory (notes Claude writes itself).

По scope, official documentation divides CLAUDE.md files into four categories (loaded from broadest to most specific):

- **Уровень организационной политики**: Centrally managed by IT/DevOps (for example, `/etc/claude-code/CLAUDE.md`) for company-wide standards.
- **Пользовательский уровень `~/.claude/CLAUDE.md`**: Personal preferences and rules that apply across projects.
- **Уровень проекта `./CLAUDE.md` or `./.claude/CLAUDE.md`**: The project-level system of record for architecture, technology stack, and verification commands, shared with the repository.
- **Локальный уровень `./CLAUDE.local.md`**: Personal preferences within a project, usually added to `.gitignore` rather than committed.

Есть также два дополнительных механизма:

- **Загрузка по запросу на уровне подкаталогов**: CLAUDE.md files in subdirectories are not loaded at startup; they enter the context only when Claude reads files in that directory.
- **Авто-память**: Claude proactively writes notes based on your corrections and preferences. These notes are shared by repository, apply across worktrees, and load at most the first 200 lines or 25KB per session.

These four scopes form an **instruction hierarchy**: the official documentation says that «more specific instructions appear later in the context» (project instructions appear after user instructions). The value is that the model does not have to digest an entire giant instruction file at the beginning of every conversation; instructions are loaded close to where they apply, according to scope. This is the productized answer to Lecture 4, «Why One Giant Instruction File Fails».

## Подсистема контекста: пятиуровневый конвейер сжатия

Claude Code manages context through a **five-layer compaction pipeline**, not simply by «summarizing when it gets full». This architectural detail comes from VILA Lab's source-level analysis, [Dive into Claude Code](https://zhiqiangshen.com/projects/Claude_Code_Report/Claude_Code_Report.pdf). Lecture 5 explains how long-running tasks lose continuity; Claude Code's answer is a multi-stage funnel: first apply lossless pruning (removing redundant tool results), then perform structured distillation, and only then use lossy LLM summaries, with circuit breakers to prevent excessive compaction.

This is paired with the design of session storage: **append-oriented storage**. All history is appended to `history.jsonl`, with `/resume` recovery and fork branches. This ensures that «every session leaves a good handoff before ending» — not because the system has a good memory, but because the storage layer is append-oriented and replayable.

## Подсистема инструментов: четыре механизма расширений

Claude Code divides its extension surface into four categories, each solving a different kind of problem. This is the part of its design most worth adopting:

- **Skills**: Defined in the [official documentation](https://code.claude.com/docs/en/skills) as procedural knowledge described by `SKILL.md`, loaded automatically based on triggers through progressive disclosure. They are suited to domain knowledge about «how to do something».
- **MCP**: The JSON-RPC protocol in the [official documentation](https://code.claude.com/docs/en/mcp) connects external systems, providing a standard interface through which «the model can reach the outside world».
- **Hooks**: Deterministic scripts attached to lifecycle events such as `PreToolUse`, `PostToolUse`, and `Stop`, as described in the [official documentation](https://code.claude.com/docs/en/hooks).
- **Plugins / Subagents**: The [official documentation](https://code.claude.com/docs/en/sub-agents) describes delegating complex tasks to specialized agents.

The key design is **separation of responsibilities**: CLAUDE.md manages «what», Skills manage «how», MCP manages «where to connect», and hooks manage «when to enforce». If a team mixes these layers — for example, putting work that belongs in MCP into CLAUDE.md — it creates the context leakage described in the course.

## Обратная связь и верификация: детерминистические ограничения + разделение труда человек-агент

Lecture 10 explains that «verification only counts when the complete flow works». Claude Code implements this through two tracks:

**1. Система разрешений (детерминистические ограничения).** Claude Code's permissions do not simply «ask about everything». They combine seven modes with an ML-based classifier: low-risk operations are allowed, while high-risk operations are either confirmed or denied according to policy (for architectural details, see the [VILA Lab analysis](https://zhiqiangshen.com/projects/Claude_Code_Report/Claude_Code_Report.pdf)). This turns «setting clear task boundaries for the agent» (Lecture 7) into runtime enforcement instead of a plea in a prompt.

**2. Hooks (предотвращение преждевременных объявлений о завершении).** `PostToolUse` hooks can force checks to run after tool execution and write the results back into the context; `Stop` hooks intervene when the agent declares completion. This separates «the one doing the work» from «the one checking the work». [Anthropic explicitly observed in its harness article](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) that agents «confidently praised their work», so hooks inject **deterministic** checks instead of trusting the model's self-evaluation.

**3. Субагенты (изоляция контекста).** Each subagent's conversation history is stored in a separate sidechain file and **does not inflate the parent agent's context** (see the [VILA Lab analysis](https://zhiqiangshen.com/projects/Claude_Code_Report/Claude_Code_Report.pdf)). This combines «task boundaries» with «context isolation»: splitting a task also isolates context pollution.

## Наблюдаемость и персистентность сессий

Claude Code's logs are complete append-oriented records (history.jsonl). Together with explicit commands such as `/compact`, `/clear`, and `/init`, this lets you manage context state proactively instead of passively waiting for it to fill up. `/init` even turns «initialize before the agent starts each task» (Lecture 6) into a command. The [official documentation](https://code.claude.com/docs/en/memory) says it automatically analyzes the codebase and generates an initial CLAUDE.md containing build commands, test instructions, and engineering conventions.

## Соответствие фреймворку курса

| Подсистема | Реализация Claude Code | Оценка |
|---|---|---|
| Инструкции | Многоуровневые scope (организация/пользователь/проект/локальный) + авто-память | Многоуровневая память — эталонная реализация |
| Инструменты | Четыре типа расширений: Skills + MCP + хуки + субагенты | Чёткое разделение ответственности — основная сила |
| Среда | Настройки в проекте + settings.json | Полагается на пользователей, описывающих среду в CLAUDE.md |
| Состояние | Append-oriented хранилище сессий + пятиуровневое сжатие + resume/fork | Extremely strong; эталонная реализация для преемственности долгосрочных задач |
| Обратная связь | Классификатор разрешений + обязательные проверки через PostToolUse hooks | Превращает «предотвращение преждевременных объявлений о завершении» в детерминистический механизм |

## Дизайны, которые стоит перенять

1. **Layer instructions by scope** instead of piling them into a single file. Directory-level CLAUDE.md files are an elegant implementation of «load close to where instructions apply».
2. **Compaction should be a staged funnel**: lossless before lossy; do not start by summarizing everything.
3. **Use hooks for deterministic checks**: preventing premature declarations of completion requires runtime enforcement, not pleas in prompts.
4. **Isolate subagent context**: split the context when splitting a task so subtask results do not pollute the main loop.
5. **Make session storage append-oriented and replayable**: handoffs depend on the storage layer, not memory.

## Ссылки (оригинальные источники / исходный код)

Каждое утверждение можно проследить до оригинальных источников или исходного кода ниже, избегая пересказов:

- **Claude Code · Memory**: A fresh context for every session, the four CLAUDE.md scopes, on-demand loading by subdirectory, auto memory (200 lines / 25KB), and `/init` generation of CLAUDE.md.
  [https://code.claude.com/docs/en/memory](https://code.claude.com/docs/en/memory)
- **Claude Code · Skills / MCP / Hooks / Sub-agents**: Definitions of the four extension mechanisms and their events (PreToolUse / PostToolUse / Stop).
  [https://code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills) ｜ [https://code.claude.com/docs/en/mcp](https://code.claude.com/docs/en/mcp) ｜ [https://code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks) ｜ [https://code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents)
- **VILA Lab, Dive into Claude Code** (source-level analysis): The five-layer compaction pipeline, seven permission modes + ML classifier, sidechain subagents, and append-oriented session storage in history.jsonl.
  [https://zhiqiangshen.com/projects/Claude_Code_Report/Claude_Code_Report.pdf](https://zhiqiangshen.com/projects/Claude_Code_Report/Claude_Code_Report.pdf)
- **Anthropic, Effective harnesses for long-running agents**: The source for the claims that «reliability comes from the harness rather than the model», that agents «confidently praised their work», and that hooks should be used for verification.
  [https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- **Claude Code Full Stack Guide** (community guide to the CLAUDE.md / Skills / MCP / Subagents / Hooks layers): Supplementary reading on the separation of responsibilities among extension mechanisms.
  [https://jsmanifest.com/claude-code-full-stack-guide](https://jsmanifest.com/claude-code-full-stack-guide)

Связанные лекции: [Лекция 3 · Почему репозиторий должен стать системой записи](../lectures/why-the-repository-must-become-the-system-of-record.md) ｜ [Лекция 9 · Почему агенты объявляют о победе слишком рано](../lectures/why-agents-declare-victory-too-early.md) ｜ [Лекция 10 · Почему сквозное тестирование меняет результаты](../lectures/why-end-to-end-testing-changes-results.md)
