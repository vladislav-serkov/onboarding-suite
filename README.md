# Spring Boot Fintech — Claude Code Onboarding Suite

Набор скиллов для Claude Code, который превращает любой Spring Boot fintech-проект (BNPL, платежи, банковские микросервисы) в "Claude-ready": генерирует `CLAUDE.md` с правилами и набор проектно-специфичных скиллов под `.claude/skills/`.

## Что внутри

```
skills/
└── project-onboarding/                    # мета-скилл — точка входа
    ├── SKILL.md                           # главный prompt, описание триггеров
    ├── phases/                            # инструкции для трёх фаз онбординга
    │   ├── 01-discovery.md
    │   ├── 02-interview.md
    │   └── 03-generation.md
    ├── checks/                            # как находить паттерны и стек
    │   ├── stack-detection.md
    │   ├── conventions-detection.md
    │   ├── pattern-detection.md
    │   └── gitlab-detection.md
    └── templates/                         # шаблоны генерируемых артефактов
        ├── CLAUDE.md.template
        ├── workflow/                      # workflow-скиллы (оркестраторы)
        │   ├── feature/                   # новая фича: план (--plan-only) или полный TDD
        │   ├── update-feature/            # дополнение существующей фичи по обновлённому ТЗ
        │   ├── verify/                    # сверка ТЗ с кодом и тестами (отчёт, без правок)
        │   ├── bugfix/
        │   ├── investigate/
        │   ├── create-mr/
        │   └── pr-review-checklist/
        └── primitives/                    # primitive-скиллы (атомарные операции)
            ├── openapi-first-endpoint/
            ├── liquibase-migration/
            ├── kafka-producer-consumer/
            ├── outbox-event/
            ├── idempotency-handler/
            ├── wiremock-integration-test/
            └── scheduled-job/
```

## Архитектура: двухуровневая

**Workflow-скиллы** (что делаем) оркестрируют процесс. Они не пишут код сами, они координируют фазы и подгружают нужные примитивы по ходу.

**Primitive-скиллы** (как делаем кусочки) — атомарные операции: добавить эндпоинт, создать миграцию, обернуть в Outbox и т.д. Каждый примитив поддерживает два режима:
- `skeleton` — создать заглушку которая компилируется (для TDD-фазы Phase 4 в `feature`)
- `implement` — реализовать полностью

## Workflow-скиллы

| Скилл | Назначение | Пауза с пользователем |
|---|---|---|
| `feature` | Новая фича: ТЗ → план → acceptance-тест → TDD цикл → готовая фича. Режим `--plan-only` — только JSON-план + gaps-анализ без реализации | Только на плане (Phase 2) |
| `update-feature` | ТЗ обновилось поверх существующей фичи (добавились пункты, изменились поля). Детект existing → diff-план → реализация только нового/изменённого через TDD, без перезаписи | Только на diff-плане (Phase 2) |
| `verify` | Сверка ТЗ с кодом и тестами. Матрица "criterion → реализован / покрыт / orphan". Полезен при ревью чужого PR, аудите спринта, перед демо | Нет |
| `bugfix` | Симптом → reproduce-тест → fix без регрессий | На воспроизведении бага (Phase 2) |
| `investigate` | Симптомы → гипотезы → отчёт (без кода) | Нет |
| `create-mr` | Diff → review → tests → ветка → MR в GitLab | Нет (failure-stop при ошибках) |
| `pr-review-checklist` | Diff против правил CLAUDE.md | Нет |

`feature` vs `update-feature`: первый создаёт фичу с нуля (failure-stop если что-то существует), второй дополняет существующую (никогда не трогает existing-компоненты). Выбор скилла = заявление о намерении.

Парсинг ТЗ → JSON-план — встроенная Phase 1 в обоих скиллах (намеренное дублирование, чтобы скиллы версионировались независимо). Если нужен только план без реализации, вызывай `feature --plan-only`.

## Primitive-скиллы

| Скилл | Когда триггерится |
|---|---|
| `openapi-first-endpoint` | Добавить HTTP endpoint |
| `liquibase-migration` | Создать миграцию БД |
| `kafka-producer-consumer` | Producer/consumer для Kafka |
| `outbox-event` | Доменное событие через Outbox pattern |
| `idempotency-handler` | Идемпотентность POST endpoint'а |
| `wiremock-integration-test` | Интеграционный тест с моком внешнего сервиса |
| `scheduled-job` | `@Scheduled` с распределённым локом |

## TDD внутри `feature`

Используется строгий TDD на двух уровнях:

1. **Acceptance-тест** (один на фичу) — пишется первым, остаётся RED до конца Phase 6
2. **Unit-тесты** per компонент — классический Red → Green → Refactor цикл

Acceptance-тест служит верифицируемой спецификацией: пока он не зелёный, фича не готова.

## Как использовать

### Установка

Скопируй `skills/project-onboarding` в:
- `~/.claude/skills/` — для использования во всех проектах (рекомендую)
- `<repo>/.claude/skills/` — для конкретного репо

### Первый запуск в новом проекте

```
cd <ваш Spring Boot проект>
claude
> настрой Claude под этот проект
```

Claude найдёт скилл `project-onboarding` по описанию, запустит discovery → интервью → генерацию.

После онбординга в репо появится:
```
CLAUDE.md
.claude/
  .onboarding-context.json    # сохранённый контекст для будущих регенераций
  skills/                     # ПЛОСКО (Claude Code требование — без вложенных workflow/primitives)
    feature/SKILL.md
    bugfix/SKILL.md
    openapi-first-endpoint/SKILL.md
    ...
```

Различие workflow vs primitive — концептуальное (живёт в `description`/триггерах), а не в файловой структуре. В исходном `onboarding-suite/templates/` подпапки `workflow/` и `primitives/` сохранены как организационная разметка для шаблонов, но при генерации скиллы кладутся плоско.

### Типичные сценарии после онбординга

```
# Реализовать фичу из ТЗ
> сделай фичу из ТЗ файла docs/refund-spec.md

# Поправить баг
> bugfix: при двойном POST /refunds создаётся два возврата

# Разобраться что происходит
> investigate: на проде растёт лаг consumer'а X

# Создать MR по готовой работе
> создай МР: ветка, ревью, тесты, MR в GitLab
```

## Версионирование

Каждый сгенерированный скилл в репо проекта — самостоятельный. Обновление мета-онбординга **не перезапишет** их автоматически. Чтобы перегенерировать конкретный скилл:

```
> пересобери скилл openapi-first-endpoint в этом репо
```

Чтобы обновить CLAUDE.md без полного онбординга:

```
> обнови CLAUDE.md по текущему коду
```

## Расширение набора

Добавить новый primitive-скилл:
1. Создать `skills/project-onboarding/templates/primitives/<name>/SKILL.md.template`
2. Добавить запись в `phases/02-interview.md` — Блок 5 (выбор скиллов)
3. При желании — добавить detection в `checks/pattern-detection.md`

Добавить новый workflow-скилл — аналогично, но в `templates/workflow/`. Учти, что workflow-скиллы могут вызывать primitive'ы и других workflow'ов.

## Известные ограничения

- Стек должен быть Spring Boot. Для Node.js / Go / Python — этот набор не подходит.
- Maven более поддержан, чем Gradle (Gradle работает, но шаблоны команд под Maven).
- GitLab — только этот хост поддержан в `create-mr`. GitHub/Bitbucket потребуют адаптации.
- Java 17+. Для Java 8/11 шаблоны с text blocks (`"""..."""`) надо переписывать на string concat.
