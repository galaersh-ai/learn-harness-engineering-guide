# Разбор дизайна харнеса DeepSeek

[DeepSeek Harness](https://deepseek.com/harness) (команда `dsh`, репозиторий `deepseek-ai/deepseek-harness`) был выпущен как Developer Preview в августе 2026. Его официальное определение direct: **Agent = Model + Environment + Tools + State** — четырёхчастная комбинация модели, среды, инструментов и состояния.

Если три предыдущих разбора продукта спрашивают «как должен быть спроектирован харнес», DeepSeek Harness задаёт более радикальный вопрос: **может ли харнес стать автономной средой выполнения, независимой от конкретной модели?** Его ответ — да, и он доводит эту идею до предела. По словам [документации архитектуры](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md): *«Каждая часть продукта является плагином, включая адаптер модели, реестр инструментов, журнал сессий и сам цикл агента»*.

В этой статье мы рассматриваем три вещи в particular: ядро на основе плагинов, швы возможностей и конвейер событий, along with its strongest engineering constraint: «Model-visible means logged».

## В одном предложении

Традиционный агент для кодирования имеет структуру «LLM + фиксированный цикл агента + фиксированный набор инструментов». DeepSeek Harness имеет структуру «модель + ядро плагинов (Cordis)». Ядро отвечает только за загрузку, выгрузку, зависимости и события плагинов и **не владеет никакими конкретными возможностями агента**. По словам [документации архитектуры](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md), «There is no privileged core to patch» и «you extend dsh by mounting a plugin beside the others». Это означает, что даже сам цикл агента не является священным или неизменяемым: вы можете комбинировать модель DeepSeek, субагентов Claude Code, удалённую песочницу, пользовательскую память, пользовательский цикл и пользовательский UI в совершенно нового агента.

Это самая thorough реализация утверждения курса, что «everything outside the model weights is the harness»: если харнес независим, сделайте его собственной операционной системой.

## Архитектурное ядро 1: шов возможностей

DeepSeek Harness represents «capabilities» as Services, and divides nearly every capability into three layers:

```text
Service Definition
        ↓
Service Provider
        ↓
Consumer
```

Take the file system as an example: under `FS Service` are multiple Providers — Local FS, E2B FS, and Remote FS — all exposed upward through a consistent set of file tools. Shell, Subprocess, Sandbox, Web, LLM, and SubAgent follow the same structure. This three-layer structure is not our own summary. The [architecture documentation · Capability seams](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/capability-seams.md) says: *«a seam is a swappable capability with three roles: a Service Definition declaring the interface, a Service Provider implementing it, and a Consumer using it, commonly a model-facing tool»*.

This resolves a longstanding question in harness engineering: **should an agent depend on «concrete tools» or on «capability interfaces»?** DeepSeek Harness chooses the latter. In terms of the course, this means the «tool subsystem» is standardized as an interface. Switching Providers leaves the tool surface exposed to the model unchanged while completely replacing the environment underneath.

## Архитектурное ядро 2: конвейер событий

DeepSeek Harness is not built around a simple «LLM → tool → LLM» flow. Internally, it uses an event pipeline in which every stage is an event point that plugins can observe:

```text
turn/start → claim input → assemble（system prompt / context / tools）
  → agent/pre-step → step/start → LLM request（agent/request）→ llm/stream
  → assistant/message → tool/call
  → tools/pre-execute（permission / guard / policy / hook）
  → tools/execute → tools/post-execute → tool/result → step/end → next turn
```

(The pipeline above is adapted from the [architecture documentation · Turn flow](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md): `turn/*`, `step/*`, `user/message`, `assistant/*`, and `tool/*` are persistent session events, while `agent/pre-step`, `agent/request`, `llm/stream`, and `tools/*` are extension points that plugins can observe.)

The greatest benefit of this design is that **many features do not require modifying the agent loop itself**. Want to run a security check before a tool executes? Listen to `tools/pre-execute`. Want to add memory? Inject it at `agent/pre-step`. Want to record behavior? Subscribe to session events. Want to modify the model request? Hook into `agent/request`. Want to decide whether reasoning should continue? Listen to `agent/turn-stopping`.

Compared with Lecture 11, «making agent execution observable», DeepSeek Harness goes further: instead of merely «adding logs», it turns **every step of the loop into an event point**, allowing observability, permissions, memory, and policy to attach to the loop as listeners instead of being hard-coded into it.

## Архитектурное ядро 3: журнал событий сессии и «Model-visible means logged»

DeepSeek Harness has an **append-only Session Event Log** and establishes a powerful engineering constraint. The [architecture documentation · Session log](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/session.md) states:

> **Model-visible means logged.** Anything that reaches a model request must be reconstructable from the log, and a runtime invariant asserts it.

(Anything visible to the model must be logged. Anything that enters a model request must be reconstructable from the log, and a runtime invariant enforces this rule.)

In other words, observability is not logging added after the fact; it is a first-principles constraint of the harness. Anything that enters the model context should be logged by default. This directly echoes the closing lecture's point that «observability belongs inside the harness» and turns the append-only storage design into a principle: logs are appended, never overwritten, and session state can be replayed.

## Соответствие фреймворку курса

| Подсистема | Реализация DeepSeek Harness | Оценка |
|---|---|---|
| Инструкции | На основе плагинов; правила/навыки вводятся как плагины | Extremely flexible, но lacks a built-in convention like CLAUDE.md |
| Инструменты | Service Definition → Provider → Consumer шов возможностей | Стандартизация подсистемы инструментов, доведённая до предела |
| Среда | Sandbox/FS/Shell Providers все заменяемые (включая удалённый E2B) | Среда полностью подключаемая |
| Состояние | Append-only журнал событий сессии + Model-visible means logged | Наблюдаемость — ограничение первого принципа |
| Обратная связь | Permission / guard / policy / hook на tools/pre-execute | Механизмы обратной связи основаны на событиях |

Фундаментальное различие между DeepSeek Harness и тремя другими продуктами в том, что Pi, Claude Code и Codex все оптимизируют харнес «inside a specific agent», while DeepSeek Harness defines the harness as an **operating system independent of the model**, with the agent itself merely a replaceable application running on that OS. The tradeoff is equally clear: greater flexibility means higher configuration cost, an inherent downside of the «harness as OS» design (the Developer Preview is also positioned as an early look at mechanisms that are still evolving).

## Дизайны, которые стоит перенять

1. **Turn every step of the loop into an event point**: Attach permissions, memory, policies, and logs to the loop as listeners instead of hard-coding them into the loop.
2. **Standardize capability seams**: Depend on «capability interfaces» instead of «concrete tools», allowing the environment to be replaced wholesale without changing the tool surface visible to the model.
3. **Model-visible means logged**: Anything visible to the model must be logged, turning observability from a «nice-to-have» into a «first-principles constraint».
4. **Append-only session logs**: Replayable state and reliable handoffs provide an engineering guarantee that «every session leaves behind clean state».

## Ссылки (оригинальные источники / исходный код)

Каждое утверждение можно проследить до оригинальных источников или исходного кода ниже, избегая пересказов:

- **DeepSeek Harness**: The product definition «Agent = Model + Environment + Tools + State», Developer Preview positioning, and the `dsh` command.
  [https://deepseek.com/harness](https://deepseek.com/harness)
- **deepseek-ai/deepseek-harness** (command `dsh`, MIT license):
  [https://github.com/deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)
- **architecture.md**: The central source for this article — «Every part of the product is a plugin», «There is no privileged core to patch», the Turn flow event pipeline, the three roles of Capability seams, «Model-visible means logged» and its runtime invariant, the append-only Session Event Log, and capability seams such as fs/tools/telemetry and the `ctx.*` subsystems.
  [https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md)
- **Supporting architecture documentation**: An introduction to the Cordis core (plugins contribute services, typed events, reversible effects), capability seam details, and the Session subsystem.
  [https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.md) ｜ [https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/capability-seams.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/capability-seams.md) ｜ [https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/session.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/session.md)

Связанные лекции: [Лекция 11 · Почему наблюдаемость belongs inside the harness](../lectures/why-observability-belongs-inside-the-harness.md) ｜ [Лекция 12 · Почему каждая сессия must leave a clean state](../lectures/why-every-session-must-leave-a-clean-state.md) ｜ [Лекция 2 · Что такое харнес на самом деле](../lectures/02-what-a-harness-actually-is.md)
