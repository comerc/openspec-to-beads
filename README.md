# Upgrade: OpenSpec и Beads в Cursor

Разработка с ИИ-ассистентами часто напоминает поездку с талантливым, но забывчивым штурманом. Он отлично знает карту (код), но постоянно забывает пункт назначения (бизнес-задачу) и пройденный маршрут (контекст).

Мы привыкли работать в режиме "Prompt & Pray": написали длинный промпт, получили код, внесли правки. Но на дистанции сложной фичи контекст размывается. Агент начинает галлюцинировать, терять детали или переписывать одно и то же. Проблема не в модели, а в отсутствии **долгосрочной памяти** и **четкого контракта**.

В этой статье я расскажу, как превратить хаотичный диалог с Cursor в структурированный инженерный процесс. Мы объединим два инструмента:

1.  **OpenSpec** - чтобы зафиксировать "что и зачем мы делаем" (Spec-Driven Development).
2.  **Beads** - чтобы управлять тем, "как и в каком порядке" это выполнять (граф задач).
3.  **Cursor** - как среду, которая связывает их воедино.

Если вы устали от того, что ИИ теряет нить повествования на середине рефакторинга, этот подход для вас.

## Проблема контекста

Когда мы даем задачу живому разработчику, мы не диктуем ему каждую строчку. Мы даем ТЗ (спецификацию) и ожидаем, что он сам разобьет работу на этапы. С ИИ мы часто пропускаем этот ритуал, сразу требуя код.

OpenSpec и Beads решают эту проблему, перенося менеджмент задач внутрь репозитория. Это не внешняя Jira, которую ИИ не видит. Это файлы, которые лежат рядом с кодом и являются единственным источником правды.

## OpenSpec: Контракт вместо чата

OpenSpec - это фреймворк, который заставляет вас (и агента) сначала планировать, а потом делать. Вместо того чтобы сразу бросаться в реализацию, мы создаем "Change" - папку с описанием изменений.

Внутри Change живут три главных файла:

- `proposal.md` - бизнес-контекст. Зачем это нужно?
- `tasks.md` - верхнеуровневый план.
- `spec.md` - техническая спецификация. Как это должно работать? Какие форматы данных? Какие ограничения?

Ценность OpenSpec в том, что он **отделяет намерение от реализации**. Спецификация становится контрактом. Пока контракт не утвержден (вами), код писать рано. А когда код написан, мы можем автоматически проверить, соответствует ли он контракту.

В Cursor это работает нативно через команды `/openspec-proposal`, `/openspec-apply`. Агент сам генерирует структуру, вы ее правите, и только после согласования начинается работа.

## Beads: Графовая память агента

Обычный список TODO плох тем, что он линейный. Реальная разработка - это граф. "Нельзя делать фронтенд, пока не готова ручка API". "Нельзя мержить, пока не пройдены тесты".

Beads (разработка того самого Стива Йегги) - это инструмент, который хранит задачи в виде графа зависимостей прямо в папке `.beads/`.

Почему это важно для ИИ:

1.  **Состояние**: Агент точно знает, какие задачи `ready` (можно брать), а какие `blocked` (ждут зависимости).
2.  **Изоляция**: Агент берет одну маленькую задачу, делает ее, закрывает (`bd close`) и забывает. Ему не нужно держать в контексте весь проект.
3.  **Воспроизводимость**: Вся история работы сохраняется в git. Вы можете в любой момент посмотреть, почему было принято то или иное решение.

---

## Настройка среды

Чтобы повторить этот workflow, вам понадобится установить инструменты и подготовить проект.

### Установка (один раз на машину)

```bash
# OpenSpec
npm install -g @fission-ai/openspec@latest

# Beads
npm install -g @beads/bd@latest
```

### Инициализация в проекте

Заходим в корень вашего проекта:

```bash
# Инициализация OpenSpec
openspec init

# Инициализация Beads
bd init
```

OpenSpec попросит команду в чате: "Please read openspec/project.md and help me fill it out with details about my project, tech stack, and conventions".

Теперь у вас есть инфраструктура. Но чтобы Cursor научился связывать эти инструменты, нам нужно добавить ему кастомные инструкции.

### Настройка правил для агента

Отредактируйте файл `AGENTS.md` в корне проекта. Это "системный промпт", который Cursor читает перед каждым ответом. (Конечно, тут есть свобода для творчества.)

<details>
<summary>Показать полный текст AGENTS.md</summary>

```markdown
<!-- OPENSPEC:START -->

# OpenSpec Instructions

These instructions are for AI assistants working in this project.

Always open `@/openspec/AGENTS.md` when the request:

- Mentions planning or proposals (words like proposal, spec, change, plan)
- Introduces new capabilities, breaking changes, architecture shifts, or big performance/security work
- Sounds ambiguous and you need the authoritative spec before coding

Use `@/openspec/AGENTS.md` to learn:

- How to create and apply change proposals
- Spec format and conventions
- Project structure and guidelines

Keep this managed block so 'openspec update' can refresh the instructions.

<!-- OPENSPEC:END -->

# Unified Workflow

We operate in a cycle: **OpenSpec (What) → Beads (How) → Code (Implementation)**.

## 1. Intent Formation

The user initiates with:
`/openspec-proposal "Add 2FA authentication"`

OpenSpec creates a change folder (`openspec/changes/<change-id>/`) containing:

- `proposal.md`: Business value and scope.
- `tasks.md`: High-level task list.
- `design.md`: Technical design (optional).
- `specs/.../spec.md`: Requirements and acceptance criteria.

**Agent Goal**: Edit these files until they represent a signable contract.

**DO NOT proceed to step 2 until you are explicitly told the keyword "Go!" in English.**

## 2. Task Transformation

Once the change is approved, execute the agent command:
`/openspec-to-beads <change-id>`

The agent must:

1.  Read the change files.
2.  Create a Beads Epic for the feature. Include a short description summarizing the intent and referencing the change folder (e.g., "See openspec/changes/<change-id>/").
3.  Create Beads Tasks for each item in `tasks.md`. Include a brief description for each task to provide context (why this issue exists and what needs to be done).
4.  Set dependencies (e.g., Infra blocks Backend blocks Frontend).

Result: A **live task graph in `.beads/`**, not just text.

## 3. Execution

Work loop:

- `bd ready`: Check actionable tasks
- `bd show <task-id>`: Get task context
- Implement code
- `bd close <task-id>`: Complete task
- `bd sync`: Sync state

**Rule**: Only work on tasks listed in `bd ready`.

## 4. Fixation

When all tasks are complete, execute the agent commands:

- `/openspec-apply <change-id>`: Verify code meets specs.
- Then, when ready,
- `/openspec-archive <change-id>`: Archive the change.

---

## Agent Mental Checklist

1.  **Start**: Is there an active OpenSpec change?
    - No? → Create one (`/openspec-proposal`).
    - Yes? → Read `proposal.md` and `tasks.md`.
2.  **Plan**: Are tasks tracked in Beads?
    - No? → Generate graph (`/openspec-to-beads`).
    - Yes? → Work from `bd ready`.
3.  **Align**: Keep OpenSpec (Intent) ↔ Beads (Plan) ↔ Code (Reality) in sync.

---

## Landing the Plane (Session Completion)

**When ending a work session**, you MUST complete ALL steps below. Work is NOT complete until `git push` succeeds.

**MANDATORY WORKFLOW:**

1. **File issues for remaining work** - Create issues for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **PUSH TO REMOTE** - This is MANDATORY:
   - `git pull --rebase`
   - `bd sync`
   - `git push`
   - `git status` - MUST show "up to date with origin"
5. **Clean up** - Clear stashes, prune remote branches
6. **Verify** - All changes committed AND pushed
7. **Hand off** - Provide context for next session

**CRITICAL RULES:**

- Work is NOT complete until `git push` succeeds
- NEVER stop before pushing - that leaves work stranded locally
- NEVER say "ready to push when you are" - YOU must push
- If push fails, resolve and retry until it succeeds

## Issue Tracking

This project uses **bd (beads)** for issue tracking.
Run `bd prime` for workflow context.

**Quick reference:**

- `bd ready` - Find unblocked work
- `bd create "Title" --type task --priority 2 --description "..."` - Create ad-hoc issue
- `bd close <task-id>` - Complete work
- `bd sync` - Sync with git (run at session end)

For full workflow details: `bd prime`
```

</details>

### Добавление кастомной команды

Чтобы агент умел одной командой превращать markdown-план в граф задач, создайте файл `.cursor/commands/openspec-to-beads.md`.

<details>
<summary>Показать код команды /openspec-to-beads</summary>

```markdown
---
name: /openspec-to-beads
id: openspec-to-beads
category: OpenSpec
description: Use this command after an OpenSpec change is approved.
---

## What to do

1. Ask the user (if not provided) which OpenSpec change ID to use.
2. Read:
   - `openspec/changes/<id>/proposal.md`
   - `openspec/changes/<id>/tasks.md`
   - any `openspec/changes/<id>/specs/**/spec.md`
3. Based on these files:
   - Create a Beads epic with `bd create "Implement <feature-name>" --type epic --priority 0 --description "<epic description>"`.
   - For each concrete implementation step in `tasks.md`, create a child task:
     - `bd create "<task title>" --type task --parent <epic-id> --priority 0 or 1 --description "<task description>"`.
   - Add dependencies using `bd dep add <child-id> <parent-id>`:
     - migrations & infra → block backend
     - backend → block UI
     - implementation → block release/docs
4. Run `bd ready` and summarize which tasks are now ready to start.
5. Print:
   - epic ID
   - all created task IDs
   - a short explanation of the dependency graph.
```

</details>

### Рестарт

И чтобы Cursor подхватил служебные файлы, вроде бы нужно его перезагрузить.

## Единый Workflow в действии

Теперь, когда всё настроено, вот как выглядит идеальный цикл разработки фичи.

### Формирование намерения

Вы пишете в чат:

```text
/openspec-proposal "Добавить 2FA авторизацию"
```

Что делает OpenSpec:

- создает change-папку в `openspec/changes/<change-id>/` с:
  - `proposal.md` - бизнес-описание (зачем / что);
  - `tasks.md` - список задач;
  - `design.md` (если нужно) - техдизайн;
  - `specs/.../spec.md` - требования и критерии приёмки.

**Ваша задача:** через Cursor отредактировать `proposal.md`, `tasks.md` и `specs/.../spec.md` до состояния "можно подписывать как контракт".

Все эти файлы можно изменить не только напрямую, но и через чат с агентом. Просто уточняете требования в диалоге, и агент сам вносит правки куда нам надо. Например: хочу минимальную реализацию "email-based OTP" In-Memory без привязки email-а к пользователю. Всё! Можно не указывать, где исправлять.

Ещё наблюдаю, что некоторые модели забывают оформить таски нумерацией и чекбоксами в `tasks.md`. Верить нельзя никому.

### Трансформация в задачи

Когда change согласован, запускаем нашу новую команду, напишите в чате:

```text
/openspec-to-beads <change-id>
```

Или просто "Go!" - это магическое заклинание. :)

**Логика работы агента:**

Агент читает файлы changes и автоматически создает структуру задач:

1. Создает эпик в Beads.
2. Для каждой задачи из `tasks.md` создает задачу в Beads.
3. Проставляет зависимости между ними.

Например:

- "Создать таблицу users" (блокирует Бэкенд)
- "API эндпоинт" (блокируется Базой, блокирует Фронтенд)
- "Форма входа" (блокируется API)

В итоге получаем **живой граф задач в `.beads/`**, а не просто текст.

### Исполнение

Дальше агент работает в цикле:

- `bd ready`: какие задачи можно брать прямо сейчас
- `bd show <task-id>`: детали задачи (описание, deps, parent)
- делаем работу в коде
- `bd close <task-id>`: закрываем задачу
- `bd sync`: синхронизация с Git

Агент смотрит список `bd ready`. Видит только то, что можно делать сейчас. Он берет задачу, выполняет её, прогоняет тесты. Пишет `bd close <task-id>`. Beads автоматически разблокирует следующие задачи.

### Фиксация

Когда все задачи закрыты, вы пишете в чат:

- `/openspec-apply <change-id>`: можно убедиться, что код соответствует спекам
- `/openspec-archive <change-id>`: заархивировать change как завершённый

OpenSpec сохраняет спеку как **историю изменения системы**. (Beads уже зафиксировал **историю работы** по ходу выполнения задач).

## А когда можно обойтись без Beads

- Solo‑проект или очень маленькая команда.
- Задачи краткоживущие (до пары дней) и немногочисленные.
- Вы и так держите картину в голове, а ИИ используете больше как "супер‑автокомплит".
- Нет требований по строгому контролю прогресса.

См. предыдущую заметку [про OpenSpec](https://habr.com/ru/articles/983062/) без Beads.

> Кодер с новой технологией - как маньяк с бензопилой, который носится по этажам психбольницы и пытается везде её применить.

## Заключение

Использование OpenSpec и Beads превращает "общение с чат-ботом" в управление виртуальным сотрудником. Вы перестаете бороться с потерей контекста. Спецификация держит рамки продукта, Beads держит порядок действий, а Cursor обеспечивает интерфейс.

Это позволяет делать сложные, многоэтапные задачи, не боясь, что к десятому сообщению агент забудет, с чего всё начиналось.

## Полезные ссылки

1.  [OpenSpec + Cursor Forum Discussion](https://forum.cursor.com/t/openspec-lightweight-portable-spec-driven-framework-for-ai-coding-assistants/134052)
2.  [OpenSpec GitHub](https://github.com/Fission-AI/OpenSpec)
3.  [Beads GitHub](https://github.com/steveyegge/beads)
4.  [Beads Best Practices (Steve Yegge)](https://steve-yegge.medium.com/beads-best-practices-2db636b9760c)
5.  [AGENTS.md Specification](https://agents.md/)
