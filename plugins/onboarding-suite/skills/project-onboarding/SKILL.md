---
name: project-onboarding
description: Используется когда пользователь хочет настроить Claude Code под Spring Boot fintech-проект. Триггеры — фразы "настрой Claude под проект", "сгенерируй CLAUDE.md", "init project", "onboard repo", "создай скиллы под проект", "обнови CLAUDE.md". Выполняет анализ кодовой базы, проводит структурированное интервью, затем генерирует CLAUDE.md и набор проектно-специфичных скиллов в .claude/skills/. Специализирован под стеки с Spring Boot, Kafka, PostgreSQL, Liquibase, OpenAPI-first, BNPL/платежи. Не для Node.js/Go/Python проектов.
---

# Project Onboarding для Spring Boot Fintech

Этот скилл превращает любой Spring Boot микросервисный проект в "Claude-ready": генерирует `CLAUDE.md` с правилами и двухуровневый набор скиллов под `.claude/skills/`.

## Когда использовать

- Пользователь пришёл в новый репо и хочет настроить Claude Code под него.
- В репо нет `CLAUDE.md` или он устарел.
- Нужно зафиксировать конвенции команды как машиночитаемые правила.
- Стек содержит Spring Boot + минимум одно из: Kafka, PostgreSQL+Liquibase, OpenAPI, WireMock.

Если стек кардинально другой (Node.js, Go, Python) — этот скилл не подходит, скажи об этом пользователю и не запускай discovery.

## Что генерируется

В целевом репо после онбординга:

```
CLAUDE.md                                 # правила проекта
.claude/
  .onboarding-context.json                # сохранённый контекст
  skills/                                 # ВСЕ скиллы плоско (Claude Code требование)
    feature/SKILL.md                      # workflow: новая фича TDD
    update-feature/SKILL.md               # workflow: дополнить фичу
    verify/SKILL.md                       # workflow: сверка ТЗ vs код
    bugfix/SKILL.md                       # workflow
    investigate/SKILL.md                  # workflow
    create-mr/SKILL.md                    # workflow
    pr-review-checklist/SKILL.md          # workflow
    openapi-first-endpoint/SKILL.md       # primitive
    liquibase-migration/SKILL.md          # primitive
    kafka-producer-consumer/SKILL.md      # primitive
    outbox-event/SKILL.md                 # primitive
    idempotency-handler/SKILL.md          # primitive
    wiremock-integration-test/SKILL.md    # primitive
    scheduled-job/SKILL.md                # primitive
```

**Важно:** Claude Code сканирует только плоскую структуру `.claude/skills/<name>/SKILL.md`. Не создавай подпапки `workflow/` или `primitives/` — Claude их не увидит. Различие workflow vs primitive живёт в `description` (через триггеры и явные ссылки на другие скиллы).

Какие именно скиллы генерировать — выбирается в интервью на основе обнаруженного стека. Не нашёл Kafka в discovery → не предлагаю Kafka-скиллы.

## Workflow: 3 фазы

Всегда выполняй фазы строго по порядку. Каждая фаза детализирована в отдельном файле.

### Фаза 1 — Discovery (двухэтапная)
Файл: `phases/01-discovery.md`

- **Этап 1a (light):** топология репо, build system, наличие Claude-артефактов. Без чтения исходников.
- **Этап 1b (deep):** точечный анализ только релевантных подсистем (Kafka, Liquibase, OpenAPI, тесты, GitLab CI, TDD-инфраструктура).

Результат — внутренний JSON `project-context` со всем найденным контекстом.

### Фаза 2 — Interview (блоки по темам)
Файл: `phases/02-interview.md`

5 тематических блоков, в каждом 3-5 вопросов. Спрашивай **только то, что нельзя узнать из кода**. Если в discovery не нашли подсистему — пропускай блок.

### Фаза 3 — Generation
Файл: `phases/03-generation.md`

Генерация CLAUDE.md и выбранных скиллов с подстановкой реальных значений из context.

## Helper-документы

- `checks/stack-detection.md` — определение стека по pom.xml/build.gradle
- `checks/conventions-detection.md` — поиск naming patterns (модули, SQL префиксы, Kafka топики)
- `checks/pattern-detection.md` — Outbox, идемпотентность, 3DS2 маркеры
- `checks/gitlab-detection.md` — обнаружение GitLab настроек (для create-mr)

## Жёсткие правила

1. **Не перезаписывай существующие файлы без подтверждения.** Если `CLAUDE.md` уже есть — покажи diff и спроси: merge, replace или skip.
2. **Никогда не выдумывай конвенции.** Если в коде паттерн не найден — спроси в интервью или оставь TODO в шаблоне.
3. **Подставляй реальные значения.** Если SQL префикс проекта `flp_baas_facade` — везде в скиллах должен быть `flp_baas_facade`, не дефолтный `app_schema`.
4. **Минимизируй вопросы.** Перед каждым вопросом проверь — точно ли это нельзя извлечь из кода.
5. **Сохрани discovery-артефакт.** В конце фазы 1 запиши `.claude/.onboarding-context.json` — это позволит при пересборке скиллов не делать discovery заново.
6. **Workflow-скиллы вызывают primitive'ы как sub-skills.** Документируй это явно в каждом workflow-шаблоне через секцию "Используемые примитивы".

## Регенерация

Если пользователь говорит "пересобери скилл X" или "обнови CLAUDE.md":
1. Прочитай `.claude/.onboarding-context.json`.
2. Если контекст устарел (> 30 дней или пользователь говорит "много изменилось") — запусти discovery заново.
3. Перегенерируй только запрошенный артефакт.
