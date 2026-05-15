# Фаза 3: Generation

Создай файлы в целевом репо на основе `project-context`.

## Порядок генерации

1. Проверь существующие файлы (`CLAUDE.md`, `.claude/skills/*`). Если что-то есть — покажи diff и спроси: merge / replace / skip.
2. Создай `CLAUDE.md` из `templates/CLAUDE.md.template`.
3. Для каждого скилла из `team_decisions.selected_skills` создавай **плоско** в `.claude/skills/<name>/SKILL.md`:
   - Workflow → `.claude/skills/<name>/SKILL.md` из `templates/workflow/<name>/SKILL.md.template`
   - Primitive → `.claude/skills/<name>/SKILL.md` из `templates/primitives/<name>/SKILL.md.template`

   **ВАЖНО:** Claude Code сканирует skills плоско — подпапки `workflow/`/`primitives/` под `.claude/skills/` НЕ обнаруживаются автоматически. Разделение workflow vs primitive живёт только в frontmatter `description` каждого скилла, не в файловой структуре.
4. Сохрани финальный `.claude/.onboarding-context.json`.
5. Покажи пользователю summary с tree созданных файлов и список TODO.

## Подстановка значений

В шаблонах используются плейсхолдеры `{{ key.subkey }}`. Замени их перед записью файла.

| Плейсхолдер | Источник | Пример |
|---|---|---|
| `{{ stack.spring_boot }}` | discovery | `3.2.1` |
| `{{ stack.java_version }}` | discovery | `17` |
| `{{ topology.module_naming }}` | discovery | `bnpl-*` |
| `{{ conventions.sql_schema }}` | discovery | `flp_baas_facade` |
| `{{ conventions.changeset_naming }}` | discovery | `YYYYMMDD-NN-description` |
| `{{ conventions.http_path_template }}` | discovery | `/api/v1/{domain}` |
| `{{ conventions.kafka_topic_template }}` | discovery | `pay-later.{service}.{domain}.{action}.{direction}.queue` |
| `{{ conventions.package_root }}` | discovery | `ru.mts.bnpl` |
| `{{ libs.wiremock }}` | discovery | `3.5.4` |
| `{{ commands.build }}` | interview | `./mvnw clean install` |
| `{{ commands.test }}` | interview | `./mvnw verify` |
| `{{ commands.local_env }}` | interview | `docker-compose up -d` |
| `{{ team_decisions.lombok_policy }}` | interview | `DTOs only` |
| `{{ git.default_branch }}` | discovery | `develop` |
| `{{ git.project_path }}` | discovery | `fintech/bnpl-gateway` |
| `{{ team_decisions.branch_naming }}` | interview | `feature/BNPL-NNNN-description` |
| `{{ team_decisions.commit_convention }}` | interview | `conventional` |

Если значение **не найдено** в context — оставь TODO с пояснением:
```markdown
TODO: уточнить SQL префикс схемы (не нашёл в миграциях, не уточнили в интервью)
```

## Жёсткие правила генерации

1. **Никогда не вставляй фиктивные имена.** Если не знаешь имя модуля для примера — пиши `<your-module>` с пометкой "замени на реальное", а не выдумывай "user-service".
2. **Версии должны совпадать.** Если в pom.xml WireMock `3.5.4`, а в шаблоне теста `3.10.0` — это баг шаблона. Обнови.
3. **Не дублируй знания.** Если правило уже в CLAUDE.md, в скилле просто ссылайся: "следуй naming convention из CLAUDE.md §4".
4. **Каждый скилл — самодостаточен.** Должен работать даже без CLAUDE.md (но с отсылкой к нему).
5. **Workflow-скиллы должны явно ссылаться на используемые primitive'ы.** Каждый workflow содержит секцию "Используемые примитивы".

## Финальный output для пользователя

```
✅ Создано:
  CLAUDE.md
  .claude/
    .onboarding-context.json
    skills/
      feature/SKILL.md
      update-feature/SKILL.md
      verify/SKILL.md
      bugfix/SKILL.md
      investigate/SKILL.md
      create-mr/SKILL.md
      pr-review-checklist/SKILL.md
      openapi-first-endpoint/SKILL.md
      liquibase-migration/SKILL.md
      kafka-producer-consumer/SKILL.md
      outbox-event/SKILL.md
      idempotency-handler/SKILL.md
      wiremock-integration-test/SKILL.md
      scheduled-job/SKILL.md

📋 Что дальше:
  1. Проверь CLAUDE.md — нашёл N TODO которые требуют твоего внимания
  2. Запусти тестовый промпт: "сделай фичу: добавь endpoint POST /api/v1/payments/refund"
     — должен сработать workflow `feature` с TDD-циклом
  3. Если что-то не так — отредактируй скилл руками или скажи "пересобери скилл X"

⚠️ TODO которые требуют твоего внимания:
  - CLAUDE.md:42 — не нашёл команду запуска интеграционных тестов
  - skills/primitives/kafka-producer-consumer/SKILL.md:18 — DLQ naming не подтверждён
  - skills/workflow/create-mr/SKILL.md:25 — glab не установлен, fallback на curl
```

## Регенерация отдельного артефакта

Если пользователь говорит "пересобери скилл X" или "обнови CLAUDE.md":
1. Прочитай `.claude/.onboarding-context.json`.
2. Если контекст устарел (>30 дней или пользователь говорит "много изменилось") — запусти discovery заново.
3. Перегенерируй только запрошенный артефакт.
