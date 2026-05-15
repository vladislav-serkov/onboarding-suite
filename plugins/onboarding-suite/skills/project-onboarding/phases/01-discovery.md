# Фаза 1: Discovery

Двухэтапный анализ кодовой базы. Цель — собрать `project-context` со всем, что можно извлечь автоматически, чтобы интервью было максимально точечным.

## Этап 1a — Light scan (быстрый, без чтения исходников)

Цель: за минимальное число операций понять топологию репо.

```bash
# 1. Корневая структура
ls -la

# 2. Модули (если монорепо)
find . -maxdepth 3 -name "pom.xml" -not -path "*/target/*" -not -path "*/node_modules/*"
find . -maxdepth 3 -name "build.gradle*" -not -path "*/build/*"

# 3. Существующие Claude-артефакты
ls -la CLAUDE.md .claude/ 2>/dev/null
find . -maxdepth 4 -name "CLAUDE.md" -not -path "*/target/*"

# 4. Build files (только корневые)
cat pom.xml 2>/dev/null | head -200
cat build.gradle build.gradle.kts settings.gradle settings.gradle.kts 2>/dev/null

# 5. GitLab признаки
ls -la .gitlab-ci.yml .gitlab/ 2>/dev/null
cat .git/config 2>/dev/null | grep -E "url\s*="
```

Заполни `project-context` после этапа 1a:

```json
{
  "build_system": "maven|gradle",
  "is_monorepo": true,
  "modules": [
    {"name": "bnpl-gateway-api", "path": "./bnpl-gateway-api"}
  ],
  "module_naming_pattern": "bnpl-*",
  "java_version": "17",
  "spring_boot_version": "3.2.1",
  "git_host": "gitlab|github|bitbucket|unknown",
  "git_remote_url": "...",
  "existing_claude_md": false,
  "existing_skills": []
}
```

Если `existing_claude_md: true` — прочитай его до начала интервью, не перезаписывай вслепую.

## Этап 1b — Deep scan (точечно по подсистемам)

Запускай каждую секцию **только если в этапе 1a нашёл маркеры**.

### 1b.1 — Стек и зависимости

См. `checks/stack-detection.md`. Извлеки версии: kafka, liquibase, wiremock, rest-assured, openapi-generator, mapstruct, lombok, testcontainers, springdoc.

### 1b.2 — Naming conventions

См. `checks/conventions-detection.md`. Извлеки:
- Корневой Java пакет
- SQL schema prefix
- Liquibase changeset naming pattern
- HTTP path template
- Kafka topic naming pattern

### 1b.3 — Архитектурные паттерны

См. `checks/pattern-detection.md`. Маркеры для:
- Outbox pattern
- Идемпотентность
- 3DS2 / платёжные потоки
- Locking patterns (`@Modifying`, `FOR UPDATE`, ReentrantLock)
- Circuit breakers (Resilience4j)

### 1b.4 — Тесты и TDD-инфраструктура

```bash
# Какие тестовые фреймворки
grep -h -E "(junit|wiremock|testcontainers|mockito|rest-assured|assertj)" \
  $(find . -name "pom.xml" -not -path "*/target/*") | sort -u

# Стиль настройки WireMock
grep -r "WireMockServer\|@AutoConfigureWireMock\|@WireMockTest" --include="*.java" -l | head -3

# Testcontainers (нужны для acceptance тестов в feature)
grep -r "@Testcontainers\|PostgreSQLContainer\|KafkaContainer" --include="*.java" -l | head

# Тестовые конфиги
find . -path "*/src/test/resources/application*.yaml" -o -path "*/src/test/resources/application*.yml" 2>/dev/null
find . -path "*/src/test/resources/application*.properties" 2>/dev/null

# Slice-тесты (показатель быстрого feedback loop)
grep -r "@DataJpaTest\|@WebMvcTest\|@JsonTest" --include="*.java" -l | head
```

Определи:
- Embedded vs External WireMock.
- Версия WireMock client должна совпадать с server.
- Используются ли Testcontainers (нужны для `feature` workflow с acceptance тестами).
- Есть ли тестовые `application*.yaml` (нужны для интеграционных тестов).

Заполни:
```json
"tdd_infrastructure": {
  "junit_version": "5",
  "uses_testcontainers": true,
  "has_test_application_yaml": true,
  "uses_slice_tests": true,
  "wiremock_style": "embedded|external|none"
}
```

Если `uses_testcontainers: false` — отметь как "TDD-blocker" для `feature` workflow и подними в интервью.

### 1b.5 — GitLab-инфраструктура (для create-mr)

См. `checks/gitlab-detection.md`. Извлеки:
- GitLab project ID или path (из remote URL)
- `.gitlab-ci.yml` — какие джобы должны быть зелёными
- `.gitlab/merge_request_templates/` — наличие шаблонов MR
- Базовая ветка из CI (`develop`, `main`, `master`)
- Branch naming pattern (если найдётся в .gitlab-ci.yml через rules)

### 1b.6 — Команды разработки

```bash
ls mvnw mvnw.cmd 2>/dev/null
ls docker-compose*.yml Dockerfile 2>/dev/null
cat docker-compose*.yml 2>/dev/null | grep -E "(postgres|kafka|redis|zookeeper)"
grep -r "spring.profiles\|@Profile" --include="*.java" --include="*.yaml" --include="*.properties" -h | sort -u | head
```

## Финальный project-context

```json
{
  "stack": {
    "build_system": "maven",
    "java_version": "17",
    "spring_boot": "3.2.1",
    "kotlin": false
  },
  "topology": {
    "is_monorepo": true,
    "modules": [...],
    "module_naming": "bnpl-*"
  },
  "conventions": {
    "package_root": "ru.mts.bnpl",
    "sql_schema": "flp_baas_facade",
    "changeset_naming": "YYYYMMDD-NN-description",
    "http_path_template": "/api/v1/{domain}",
    "kafka_topic_template": "pay-later.{service}.{domain}.{action}.{direction}.queue"
  },
  "patterns": {
    "outbox": true,
    "idempotency": true,
    "3ds2": true,
    "openapi_first": true,
    "circuit_breaker": false
  },
  "libs": {
    "wiremock": "3.5.4",
    "rest_assured": "5.4.0",
    "liquibase": "4.25.0",
    "openapi_generator": "7.2.0"
  },
  "tdd_infrastructure": {
    "uses_testcontainers": true,
    "has_test_application_yaml": true,
    "uses_slice_tests": true,
    "wiremock_style": "embedded"
  },
  "git": {
    "host": "gitlab",
    "project_path": "fintech/bnpl-gateway",
    "default_branch": "develop",
    "has_mr_template": true,
    "has_ci": true
  },
  "dev_env": {
    "has_docker_compose": true,
    "external_deps": ["postgres", "kafka"]
  },
  "claude_artifacts": {
    "has_claude_md": false,
    "existing_skills": []
  }
}
```

Сохрани этот JSON в `.claude/.onboarding-context.json` в целевом репо в конце фазы.

## Что показать пользователю в конце фазы 1

Краткое summary в чате (не весь JSON):

> **Что я нашёл:**
> - Maven monorepo, 7 модулей с префиксом `bnpl-*`
> - Spring Boot 3.2.1, Java 17
> - PostgreSQL + Liquibase, схема `flp_baas_facade`
> - Kafka (топики формата `pay-later.*.queue`)
> - OpenAPI-first (нашёл спеки в `openapi/`)
> - Outbox pattern, идемпотентность, 3DS2
> - WireMock 3.5.4 (embedded), REST Assured, AssertJ
> - Testcontainers — есть, slice-тесты используются ✓ (TDD-готов)
> - GitLab, default branch `develop`, есть MR template
>
> Перехожу к интервью — 5 коротких блоков.

Если есть TDD-blocker'ы — отметь явно:

> ⚠️ Не нашёл Testcontainers и тестовых application.yaml. Acceptance-тесты в `feature` workflow потребуют доустановки этого. Уточню в интервью.
