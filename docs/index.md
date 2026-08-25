# Learn Harness Engineering — Руководство на русском

## О курсе

**Learn Harness Engineering** — курс, посвящённый инженерии AI-агентов для написания кода. Мы глубоко изучили и обобщили наиболее передовые теории и практики Harness Engineering в индустрии.

### Основные источники

- [OpenAI: Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)
- [Anthropic: Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Anthropic: Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-for-long-running-application-development)
- [Awesome Harness Engineering](https://github.com/anthropics/awesome-harness-engineering)

### Что вы узнаете

Через систематическое проектирование окружения, управление состоянием, верификацию и системы контроля этот курс научит вас делать инструменты агентного кодирования (такие как Codex и Claude Code) по-настоящему надёжными. Курс поможет вам создавать функции, исправлять ошибки и автоматизировать задачи разработки, ограничивая вашего AI-ассистента явными правилами и границами.

### Ключевые концепции

- Ограничение поведения агента (agent) явными правилами и границами
- Поддержание контекста (context) в длительных, многосессионных задачах
- Предотвращение преждевременного объявления агентом задачи завершённой
- Верификация работы с помощью полнопайплайновых тестов и саморефлексии
- Обеспечение наблюдаемости и отладки среды выполнения

---

## Лекции

| № | Название | Описание |
|---|----------|----------|
| 1 | [Сильные модели не означают надёжного выполнения](lectures/01-why-capable-agents-still-fail.md) | Теория Harness Engineering — почему даже лучшие модели терпят неудачу |
| 2 | [Что такое Harness на самом деле](lectures/02-what-a-harness-actually-is.md) | Точное определение: пять подсистем harness |
| 3 | [Репозиторий как единый источник истины](lectures/03-why-repository-system-of-record.md) | Почему вся информация должна жить в репозитории |
| 4 | [Почему один гигантский файл инструкций не работает](lectures/04-why-one-giant-instruction-file-fails.md) | Эффект «потерянности в середине» и разбиение инструкций |
| 5 | [Почему длительные задачи теряют непрерывность](lectures/05-why-long-running-tasks-lose-continuity.md) | Сохранение состояния между сессиями |
| 6 | [Почему инициализации нужна своя фаза](lectures/06-why-initialization-needs-own-phase.md) | Выделенная фаза запуска перед работой |
| 7 | [Почему агенты берутся за слишком много и не доделывают](lectures/07-why-agents-overreach-under-finish.md) | WIP=1 и доказательства завершения |

---

## Как использовать это руководство

1. **Начните с лекций** — они дают теоретическую базу
2. **Практикуйтесь** — применяйте концепции в своих проектах
3. **Используйте шаблоны** — AGENTS.md, feature_list.json, PROGRESS.md

## Быстрый старт

Создайте файл `AGENTS.md` в корне вашего репозитория:

```markdown
# AGENTS.md

## Обзор проекта
Python 3.11 FastAPI backend, PostgreSQL 15.

## Быстрый старт
- Установка: `make setup`
- Тесты: `make test`
- Полная проверка: `make check`

## Жёсткие ограничения
- Все API должны использовать OAuth 2.0 аутентификацию
- Все запросы к БД должны использовать SQLAlchemy 2.0
- Все PR должны проходить pytest + mypy --strict + ruff check
```

Этого уже достаточно, чтобы значительно улучшить работу агента.
