# Разбор дизайна харнеса Codex

[Codex](https://openai.com/index/harness-engineering/) от OpenAI, возможно, является продуктом, наиболее глубоко связанным с «основами харнеса» среди этих четырёх. Статья, которая дала название всему направлению, Harness Engineering, сама была обобщением опыта команды OpenAI по созданию продукта с Codex. Разбор дизайна харнеса Codex therefore largely means examining the engineering practices behind that article.

Философию Codex можно обобщить в одном предложении: **репозиторий — это система записи, AGENTS.md — только справочная страница, а инженерная ценность lies in designing environments, expressing intent, and building feedback loops.**

## В одном предложении

Команда OpenAI использовала Codex для поставки продукта, который в итоге вырос до более чем миллиона строк кода за несколько недель, и **каждая строка кода была написана Codex** (см. раздел «Designing for growth» статьи [Harness Engineering](https://openai.com/index/harness-engineering/)). Их опыт отвечает на вопрос: как должна быть организована система, когда роль инженера смещается от «написания кода» к «проектированию харнеса»? Сам Codex CLI — это open-source монолитный бинарник, реализованный на Rust ([github.com/openai/codex](https://github.com/openai/codex)), но его основной вклад в дизайн харнеса lies in **conventions** and **context engineering**, not flashy extension points.

## Подсистема инструкций: AGENTS.md — это справочная страница, а не энциклопедия

Это самый влиятельный вклад Codex в теорию харнесов:

> Один гигантский файл инструкций трудно проверить механически на покрытие, актуальность, ownership и перекрёстные ссылки, поэтому дрейф неизбежен. Поэтому мы перестали рассматривать AGENTS.md как энциклопедию и начали рассматривать его как **справочную страницу**. Знания о кодовой базе живут в структурированной документации, а AGENTS.md указывает на неё.

(Пассаж выше является прямым парафразом раздела «AGENTS.md should be a directory page» статьи [Harness Engineering](https://openai.com/index/harness-engineering/).)

Лекция 4 говорит, что «один гигантский файл инструкций терпит неудачу», и Codex даёт прямой ответ: держите AGENTS.md примерно в 100 строк (оригинал рекомендует около 100 строк и перемещение содержимого в `docs/` по мере приближения к лимиту). Если не помещается, разделите на каталог `docs/` и позвольте агенту читать файлы по запросу. Это авторитетный источник для «дать карту, а не руководство».

Сопутствующий принцип — **применять инварианты, а не микроменеджмент реализации** (в оригинале: «don't micromanage the implementation; focus on invariants»). AGENTS.md should contain only inviolable constraints and verification commands, leaving the implementation details to the model. This maps directly to Lecture 2's «constraints rather than micromanagement».

## Подсистема контекста: Write-Select-Compress-Isolate

Инженерия контекста Codex может быть обобщена как четыре стратегии. Этот фреймворк был разработан сообществом после того, как «context engineering» emerged as a distinct discipline and then mapped back onto Codex (см. [Context Engineering for Codex CLI](https://codex.danielvaughan.com/2026/06/10/context-engineering-codex-cli-write-select-compress-isolate-june-2026/) для фреймворка):

- **Write**: Persist context outside the window — write conclusions into documentation and state into files instead of leaving them in the conversation. This corresponds to «the repository as the system of record».
- **Select**: Pull only the necessary tokens into the window — let AGENTS.md point the way and read files on demand instead of loading the whole repository.
- **Compress**: Preserve what truly matters — Codex supports automatic compaction and manual `/compact`, with a customizable `compact_prompt` (see [Context Engineering for Codex CLI](https://codex.danielvaughan.com/2026/06/10/context-engineering-codex-cli-write-select-compress-isolate-june-2026/)).
- **Isolate**: Divide context across different boundaries — use subagents to isolate the context of different tasks, so a frontend subagent never sees the backend database schema.

Codex also has a fine-grained environment-context design. Community source analysis in [codex-harness-internals](https://github.com/AlexKenbo/codex-harness-internals) shows that `build_environment_update_item` outputs only **changed fields** (CWD, git branch, file system) when the environment changes, rather than pasting the full system context into every turn. This is an engineering detail that avoids keeping duplicate tokens in the context.

## Инструменты и границы: изоляция worktree + субагенты

Codex has two core harness mechanisms:

**1. Изоляция среды с git worktrees.** The «Environment» section of [Harness Engineering](https://openai.com/index/harness-engineering/) states that every task runs in an independent git worktree, paired with a local observability stack (logs, metrics, and traces), so every change can be verified in an isolated environment. This is the physical implementation of Lecture 7's «setting clear boundaries for every agent task»: the boundary is enforced through environment isolation, not requested through instructions. Here, the environment subsystem becomes hard isolation.

**2. Субагенты на уровне ядра.** Codex's `spawn_agent` / `wait_agent` are core tools: the model explicitly creates a subagent, gives it independent session history and a tool set, and waits for the result. Subagents inherit AGENTS.md instructions from their parent, but run in **their own context**. Configuration lives in `.codex/agents/*.toml`, where different models and instructions can be specified (for details, see the Sub-agents section of [Context Engineering for Codex CLI](https://codex.danielvaughan.com/2026/06/10/context-engineering-codex-cli-write-select-compress-isolate-june-2026/)). This is a direct implementation of «context isolation» and also embodies the spirit of Lecture 12 on handoffs: every subagent is a work unit with clear boundaries.

## Подсистема обратной связи: вставьте команды верификации в спецификацию

One of the strongest emphases in OpenAI's practice is to list verification commands explicitly in AGENTS.md, making «how to confirm the work is correct» part of the repository. In the Codex engineering workflow, tests, CI, documentation, and observability configuration are all generated by Codex, and every one of them provides an «executable verification path». The answer to powerful but unreliable models is not to hope the model behaves responsibly; it is to make **verification paths a default component of the harness**.

Approval policies and plan mode provide feedback in the other direction: before high-risk operations execute, the system first produces a plan and requests approval, turning «task boundaries» and «human decision-making authority» into runtime controls.

## Соответствие фреймворку курса

| Подсистема | Реализация Codex | Оценка |
|---|---|---|
| Инструкции | AGENTS.md справочная страница + разделённый docs/ + применяемые инварианты | Качество учебника; определяет «дать карту, а не руководство» |
| Инструменты | Изоляция worktree + spawn_agent субагенты | Сильные границы, применяемые через изоляцию среды |
| Среда | Независимые worktrees + стек наблюдаемости | Изоляция worktree — его фирменная особенность |
| Состояние | Стратегия Write (состояние записывается в файлы/документацию) | Полагается на соглашения, а не на встроенную память |
| Обратная связь | Команды верификации в спецификации + политики одобрения + режим плана | Делает пути обратной связи значением по умолчанию; стоит перенять |

Контраст между Codex и Claude Code revealing. Claude Code uses «addition», putting memory, permissions, and subagents into the core. Codex uses «subtraction», keeping the core restrained and placing more responsibility on repository conventions and context engineering. This is why the community often says that «Codex's harness philosophy is more valuable than its code».

## Дизайны, которые стоит перенять

1. **Write AGENTS.md as a directory page**: Keep it to roughly 100 lines, point to details in docs/, and make it mechanically checkable.
2. **Specify only invariants; do not micromanage implementation**: Hard constraints + verification commands, with the rest left to the model.
3. **Use worktrees for environment isolation**: Enforce task boundaries through the environment, not through pleas in instructions.
4. **Send only environment-context deltas**: Output only changed fields on each turn instead of repeatedly pasting the full system context.
5. **Use subagents for context isolation**: Split the context along with the task so subtasks do not pollute the main loop.

## Ссылки (оригинальные источники / исходный код)

Каждое утверждение можно проследить до оригинальных источников или исходного кода ниже, избегая пересказов:

- **OpenAI, Harness Engineering**: The AGENTS.md directory page and roughly 100-line recommendation, enforce invariants / don't micromanage, worktree isolation + observability stack, verification commands in the specification, the million-line product case study, approval policies, and plan mode. The primary source for every core claim in this article.
  [https://openai.com/index/harness-engineering/](https://openai.com/index/harness-engineering/)
- **OpenAI, AGENTS.md** (the standard for AGENTS.md as a cross-tool convention):
  [https://openai.com/index/agents-md/](https://openai.com/index/agents-md/)
- **Codex CLI** (a monolithic binary implemented in Rust):
  [https://github.com/openai/codex](https://github.com/openai/codex)
- **Context Engineering for Codex CLI** (community): The Write-Select-Compress-Isolate framework, `/compact` and `compact_prompt`, `spawn_agent` / `wait_agent` subagents, and `.codex/agents/*.toml` configuration.
  [https://codex.danielvaughan.com/2026/06/10/context-engineering-codex-cli-write-select-compress-isolate-june-2026/](https://codex.danielvaughan.com/2026/06/10/context-engineering-codex-cli-write-select-compress-isolate-june-2026/)
- **codex-harness-internals** (community source analysis): Implementation details such as incremental environment context from `build_environment_update_item`.
  [https://github.com/AlexKenbo/codex-harness-internals](https://github.com/AlexKenbo/codex-harness-internals)

Связанные лекции: [Лекция 3 · Почему репозиторий должен стать системой записи](../lectures/why-the-repository-must-become-the-system-of-record.md) ｜ [Лекция 4 · Почему один гигантский файл инструкций терпит неудачу](../lectures/why-one-giant-instruction-file-fails.md) ｜ [Лекция 7 · Почему агенты выходят за рамки и не доделывают](../lectures/why-agents-overreach-and-under-finish.md)
