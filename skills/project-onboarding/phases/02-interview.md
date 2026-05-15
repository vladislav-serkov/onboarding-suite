# Фаза 2: Interview

5-6 тематических блоков. **Спрашивай только то, что не извлёк из кода.** Каждый блок задаётся **одним вызовом `AskUserQuestion`** с несколькими структурированными вопросами одновременно.

## Принципы

1. **AskUserQuestion, не свободный текст.** Каждый блок — один `AskUserQuestion` tool call. Свободный ввод (option `"other"`) — только там где варианты заранее не угадать.
2. **Confirm-or-change first.** Если discovery нашёл значение — первый вопрос блока всегда вида "Я нашёл X. Это правильно?" с radio `confirm` / `change`. При `confirm` — остальные вопросы блока скипаются.
3. **Адаптивность.** Если в discovery не нашёл подсистему — пропусти весь блок. Если нашёл всё — пропусти все вопросы кроме confirm.
4. **Источник ответа.** Для каждого зафиксированного значения сохраняй `source`: `discovery` / `user-confirmed` / `user-changed` / `user-text`. Это попадёт в `.onboarding-context.json` и используется при регенерации.
5. **Группами по блокам.** Не дроби блок на 5 отдельных AskUserQuestion подряд — один блок = один tool call с массивом вопросов.

## Формат AskUserQuestion для каждого блока

Каждый блок описывается как JSON-структура, которая мапится 1-в-1 на параметры `AskUserQuestion`:

```json
{
  "questions": [
    {
      "header": "Короткий заголовок (≤12 символов)",
      "question": "Полный текст вопроса с контекстом из discovery",
      "multiSelect": false,
      "options": [
        {"label": "Подтвердить", "description": "Объяснение варианта"},
        {"label": "Изменить", "description": "..."}
      ]
    }
  ]
}
```

Где `multiSelect: true` — для чекбоксов (выбор нескольких скиллов), `false` — radio.

---

## Блок 1 — Архитектурные границы

**Запускай только если** `is_monorepo: true` и модулей >= 2.

```json
{
  "questions": [
    {
      "header": "Модули",
      "question": "Я нашёл модули: {{ discovery.modules | join(', ') }}. Какие из них публичные API (gateway)?",
      "multiSelect": true,
      "options": [
        {"label": "<module-name-1>", "description": "Публичный API"},
        {"label": "<module-name-2>", "description": "..."},
        {"label": "Другое", "description": "Уточнить в свободной форме"}
      ]
    },
    {
      "header": "Off-limits",
      "question": "Есть ли модули, которые агенту запрещено трогать (legacy, чужая команда, generated)?",
      "multiSelect": true,
      "options": [
        {"label": "<module-name>", "description": "..."},
        {"label": "Нет, все доступны", "description": "Agent может править любой модуль"},
        {"label": "Другое", "description": "Свободный список"}
      ]
    },
    {
      "header": "Parent POM",
      "question": "Как добавляются новые модули?",
      "multiSelect": false,
      "options": [
        {"label": "Через parent pom", "description": "Нашёл parent в {{ discovery.parent_pom_path }}"},
        {"label": "Отдельно", "description": "Каждый модуль самостоятельный"},
        {"label": "Другое", "description": "..."}
      ]
    }
  ]
}
```

Source: `user-text` для всех ответов (свободный выбор из обнаруженных модулей).

---

## Блок 2 — Команды и окружение

**Pre-check:** если в discovery уже найдены `commands.test`, `commands.build`, `commands.lint` — confirm-or-change.

```json
{
  "questions": [
    {
      "header": "Сборка",
      "question": "Команда сборки. Нашёл: `{{ discovery.commands.build }}`. Подтвердить?",
      "multiSelect": false,
      "options": [
        {"label": "Подтвердить", "description": "{{ discovery.commands.build }}"},
        {"label": "Изменить", "description": "Указать другую команду"}
      ]
    },
    {
      "header": "Тесты",
      "question": "Есть ли отдельные профили тестов?",
      "multiSelect": true,
      "options": [
        {"label": "unit (mvn test)", "description": "Быстрые без контейнеров"},
        {"label": "integration", "description": "Профиль -P integration-test"},
        {"label": "e2e", "description": "Полный стек через docker-compose"},
        {"label": "Только один прогон", "description": "Все тесты в одном mvn verify"}
      ]
    },
    {
      "header": "Окружение",
      "question": "Как локально поднимается окружение?",
      "multiSelect": false,
      "options": [
        {"label": "docker-compose up", "description": "Нашёл docker-compose.yml"},
        {"label": "testcontainers", "description": "Поднимается из тестов автоматически"},
        {"label": "staging", "description": "Подключение к удалённому окружению"},
        {"label": "Другое", "description": "..."}
      ]
    },
    {
      "header": "Линтер",
      "question": "Линтер/форматтер — должен ли агент его гонять перед коммитом?",
      "multiSelect": false,
      "options": [
        {"label": "Да, всегда", "description": "Нашёл Spotless/Checkstyle"},
        {"label": "Только перед PR", "description": "В create-mr workflow"},
        {"label": "Нет линтера", "description": "В проекте не настроен"}
      ]
    }
  ]
}
```

Если первый вопрос → `Подтвердить` → source: `user-confirmed`, остальные подкоманды скипаются (берём из discovery).

---

## Блок 3 — Болевые точки

**Всегда задаём** (нельзя извлечь из кода). Это становится жёсткими правилами в CLAUDE.md.

```json
{
  "questions": [
    {
      "header": "Frequent fails",
      "question": "Что в этом проекте чаще всего ломается у новых разработчиков?",
      "multiSelect": true,
      "options": [
        {"label": "Forgotten outbox", "description": "Прямой kafka.send вместо OutboxService"},
        {"label": "N+1 в JPA", "description": "Fetch eager, missing @EntityGraph"},
        {"label": "Idempotency", "description": "POST без Idempotency-Key"},
        {"label": "Migration locks", "description": "ALTER TABLE на больших таблицах"},
        {"label": "PII в логах", "description": "Не замаскированные карты/телефоны"},
        {"label": "Другое", "description": "Свободный текст"}
      ]
    },
    {
      "header": "PR pain",
      "question": "Что бесит в PR-ревью больше всего? (типичные правки)",
      "multiSelect": true,
      "options": [
        {"label": "Generated DTO правлены руками", "description": "OpenAPI-first нарушен"},
        {"label": "Транзакция в controller", "description": "Должна быть в service"},
        {"label": "Mocks в integration", "description": "@MockBean вместо реальной БД"},
        {"label": "Lombok на entities", "description": "Запрещено политикой"},
        {"label": "Другое", "description": "..."}
      ]
    },
    {
      "header": "Incidents",
      "question": "Были ли недавно прод-инциденты? Корневые причины часто становятся анти-паттернами.",
      "multiSelect": false,
      "options": [
        {"label": "Да, опишу", "description": "Опишу свободно — станет anti-pattern в CLAUDE.md"},
        {"label": "Нет инцидентов", "description": "Пропустить"}
      ]
    }
  ]
}
```

---

## Блок 4 — Правила, которые важны команде

Каждый ответ → конкретный пункт в `team_decisions`.

```json
{
  "questions": [
    {
      "header": "OpenAPI-first",
      "question": "OpenAPI-first строгий? Сгенерированные DTO трогать руками нельзя?",
      "multiSelect": false,
      "options": [
        {"label": "Строгий", "description": "Generated DTO неприкосновенны, всё через spec"},
        {"label": "Мягкий", "description": "Generated можно править если очень нужно"},
        {"label": "Не используется", "description": "DTO пишутся руками"}
      ]
    },
    {
      "header": "Outbox",
      "question": "Outbox обязателен для cross-service событий?",
      "multiSelect": false,
      "options": [
        {"label": "Всегда", "description": "Любое kafka-событие — через Outbox"},
        {"label": "Кроме metrics", "description": "Fire-and-forget разрешён для метрик/телеметрии"},
        {"label": "Кроме metrics и notifications", "description": "Outbox только для бизнес-событий"},
        {"label": "Не используется", "description": "Прямой kafka.send везде"}
      ]
    },
    {
      "header": "Idempotency",
      "question": "Идемпотентность POST endpoints — обязательна?",
      "multiSelect": false,
      "options": [
        {"label": "Всех POST", "description": "Любой POST требует Idempotency-Key"},
        {"label": "Только финансовых", "description": "Платежи, возвраты, переводы"},
        {"label": "По решению автора", "description": "Per endpoint"}
      ]
    },
    {
      "header": "Lombok",
      "question": "Lombok разрешён? Где?",
      "multiSelect": false,
      "options": [
        {"label": "Везде", "description": "Включая JPA entities"},
        {"label": "Только DTO", "description": "На JPA entities — нет (риск с @Data + JPA)"},
        {"label": "Запрещён", "description": "Везде руками"}
      ]
    },
    {
      "header": "Logging",
      "question": "Требования к логированию",
      "multiSelect": true,
      "options": [
        {"label": "Structured (JSON)", "description": "logback с json-encoder"},
        {"label": "Correlation ID", "description": "MDC проброс между сервисами"},
        {"label": "Маскирование PII/PCI", "description": "Карты, телефоны, email — маскировать"},
        {"label": "Базовое логирование", "description": "SLF4J без особых требований"}
      ]
    },
    {
      "header": "Coverage",
      "question": "Минимальное тестовое покрытие",
      "multiSelect": false,
      "options": [
        {"label": "80% service-слой", "description": "Стандарт"},
        {"label": "70% всё", "description": "Включая utils"},
        {"label": "100% бизнес-логика", "description": "Service + domain"},
        {"label": "Не измеряем", "description": "Без формальных порогов"},
        {"label": "Другое", "description": "Указать число"}
      ]
    }
  ]
}
```

---

## Блок 5 — GitLab и финализация

**Запускай только если** `git.host: gitlab`.

```json
{
  "questions": [
    {
      "header": "Branch name",
      "question": "Branch naming. Нашёл существующие: {{ discovery.git.recent_branches | join(', ') }}. Шаблон?",
      "multiSelect": false,
      "options": [
        {"label": "feature/TICKET-NNNN", "description": "feature/MTSPAY-10892 (только тикет)"},
        {"label": "feature/TICKET-NNNN-desc", "description": "feature/MTSPAY-10892-refund-creation"},
        {"label": "{type}/TICKET", "description": "feature/, bugfix/, hotfix/ префиксы по типу"},
        {"label": "Другое", "description": "Свободный шаблон"}
      ]
    },
    {
      "header": "Default branch",
      "question": "Target ветка для MR. Нашёл `{{ discovery.git.default_branch }}`.",
      "multiSelect": false,
      "options": [
        {"label": "Подтвердить", "description": "{{ discovery.git.default_branch }}"},
        {"label": "Изменить", "description": "Указать другую"}
      ]
    },
    {
      "header": "Commit style",
      "question": "Commit messages",
      "multiSelect": false,
      "options": [
        {"label": "Conventional Commits", "description": "feat(scope): ..., fix(scope): ..."},
        {"label": "Ticket-prefix", "description": "MTSPAY-NNNN: описание"},
        {"label": "Свободные", "description": "Без формальных правил"}
      ]
    },
    {
      "header": "CI checks",
      "question": "Какие CI джобы должны быть зелёными перед мержем? Нашёл в .gitlab-ci.yml: {{ discovery.git.ci_jobs | join(', ') }}",
      "multiSelect": true,
      "options": [
        {"label": "<job-1>", "description": "..."},
        {"label": "<job-2>", "description": "..."},
        {"label": "Все обязательные jobs", "description": "Берём все из .gitlab-ci.yml"}
      ]
    },
    {
      "header": "glab CLI",
      "question": "Установлен ли glab и авторизован?",
      "multiSelect": false,
      "options": [
        {"label": "Да", "description": "glab auth status — OK"},
        {"label": "Нет, есть GITLAB_TOKEN", "description": "Fallback на curl"},
        {"label": "Ничего нет", "description": "create-mr попросит залогиниться при первом запуске"}
      ]
    }
  ]
}
```

---

## Блок 6 — TDD-инфраструктура (если есть blocker'ы)

**Запускай только если** `tdd_infrastructure.uses_testcontainers: false` или другой blocker.

```json
{
  "questions": [
    {
      "header": "Testcontainers",
      "question": "Не нашёл Testcontainers. Acceptance-тесты в `feature` workflow требуют PostgreSQL + Kafka в контейнерах. Готовы добавить?",
      "multiSelect": false,
      "options": [
        {"label": "Да, добавим", "description": "feature workflow создаст шаблон при первом запуске"},
        {"label": "in-memory H2 + embedded Kafka", "description": "Менее реалистично но быстрее"},
        {"label": "Реальные сервисы через docker-compose", "description": "compose стартует до тестов"},
        {"label": "Без acceptance-тестов", "description": "TDD только на unit-уровне — отметим как blocker"}
      ]
    },
    {
      "header": "Test config",
      "question": "Тестовые application.yaml существуют?",
      "multiSelect": false,
      "options": [
        {"label": "Да", "description": "src/test/resources/application-test.yaml есть"},
        {"label": "Нет", "description": "feature workflow создаст шаблон"}
      ]
    }
  ]
}
```

---

## Блок 7 — Выбор скиллов (финальный)

**Всегда задаётся.** Показываем что нашли + checkbox-выбор.

```json
{
  "questions": [
    {
      "header": "Workflows",
      "question": "Какие workflow-скиллы генерировать? Рекомендую все — они закрывают полный цикл от ТЗ до MR.",
      "multiSelect": true,
      "options": [
        {"label": "feature", "description": "Новая фича: план / TDD-цикл (РЕКОМЕНДУЮ)"},
        {"label": "update-feature", "description": "Дополнение существующей фичи (РЕКОМЕНДУЮ)"},
        {"label": "verify", "description": "Сверка ТЗ с кодом — отчёт без правок (РЕКОМЕНДУЮ)"},
        {"label": "bugfix", "description": "Reproduce-first исправление багов (РЕКОМЕНДУЮ)"},
        {"label": "investigate", "description": "Диагностика без кода (РЕКОМЕНДУЮ)"},
        {"label": "create-mr", "description": "Diff → ветка → MR в GitLab (РЕКОМЕНДУЮ если GitLab найден)"},
        {"label": "pr-review-checklist", "description": "Self-review против CLAUDE.md (РЕКОМЕНДУЮ)"}
      ]
    },
    {
      "header": "Primitives",
      "question": "Какие primitive-скиллы генерировать? Помечены ✓ те, для которых нашёл паттерны в коде.",
      "multiSelect": true,
      "options": [
        {"label": "openapi-first-endpoint", "description": "{{ discovery.found_openapi ? '✓ нашёл OpenAPI' : 'OpenAPI не найден' }}"},
        {"label": "liquibase-migration", "description": "{{ discovery.found_liquibase ? '✓ нашёл Liquibase' : 'Не найден' }}"},
        {"label": "kafka-producer-consumer", "description": "{{ discovery.found_kafka ? '✓ нашёл Kafka' : 'Не найден' }}"},
        {"label": "outbox-event", "description": "{{ discovery.found_outbox ? '✓ нашёл Outbox' : 'Не найден' }}"},
        {"label": "idempotency-handler", "description": "{{ discovery.found_idempotency ? '✓ нашёл идемпотентность' : 'Не найден' }}"},
        {"label": "wiremock-integration-test", "description": "{{ discovery.found_wiremock ? '✓ нашёл WireMock' : 'Не найден' }}"},
        {"label": "scheduled-job", "description": "Для @Scheduled с распределённым локом"}
      ]
    }
  ]
}
```

---

## Что делать с ответами

Дополни `project-context` секцией `team_decisions` **с источником каждого значения**:

```json
"team_decisions": {
  "modules_off_limits": {
    "value": ["legacy-billing"],
    "source": "user-text"
  },
  "openapi_first_strict": {
    "value": true,
    "source": "user-changed"
  },
  "outbox_required_for": {
    "value": "all cross-service events except metrics",
    "source": "user-confirmed"
  },
  "lombok_policy": {
    "value": "DTOs only, never on JPA entities",
    "source": "user-changed"
  },
  "coverage_threshold": {
    "value": "80% for service layer",
    "source": "user-confirmed"
  },
  "branch_naming": {
    "value": "feature/TICKET-NNNN",
    "source": "discovery"
  },
  "commit_convention": {
    "value": "conventional",
    "source": "user-changed"
  },
  "selected_skills": {
    "value": [
      "feature", "update-feature", "verify", "bugfix", "investigate",
      "create-mr", "pr-review-checklist",
      "openapi-first-endpoint", "liquibase-migration",
      "kafka-producer-consumer", "outbox-event",
      "idempotency-handler", "wiremock-integration-test"
    ],
    "source": "user-confirmed"
  }
}
```

### Значения `source`

| Source | Что означает |
|---|---|
| `discovery` | Значение взято из Phase 1, пользователь подтвердил через `confirm` |
| `user-confirmed` | Discovery нашёл, пользователь явно нажал "подтвердить" в AskUserQuestion |
| `user-changed` | Discovery нашёл, но пользователь выбрал другой вариант |
| `user-text` | Discovery не нашёл, пользователь ввёл свободный текст ("Другое") |
| `default` | Discovery не нашёл и пользователь пропустил вопрос — взят дефолт из шаблона |

При **регенерации** (см. `phases/03-generation.md`) `source` используется чтобы решить какие значения можно автоматически обновить из нового discovery, а какие — пользовательский выбор и нельзя трогать без подтверждения.

Запиши обновлённый context в `.claude/.onboarding-context.json` перед фазой 3.

## Жёсткие правила

1. **Один блок = один AskUserQuestion tool call.** Не дроби.
2. **Не задавай вопросы которые можно извлечь из кода.** Если в discovery нашёл — confirm-or-change, не голый вопрос.
3. **Для каждого ответа фиксируй `source`.** Без него регенерация будет принимать неправильные решения.
4. **Свободный текст ("Другое") — только когда варианты заранее не угадать.** Для commit-convention свободный текст не нужен, для modules-off-limits нужен.
5. **Не более 5 вопросов в одном AskUserQuestion.** Если блок раздут — раздели на под-блоки логически.
