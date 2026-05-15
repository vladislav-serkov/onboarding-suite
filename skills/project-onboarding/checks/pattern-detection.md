# Pattern Detection

Поиск архитектурных паттернов в коде проекта.

## Outbox Pattern

```bash
# Java маркеры
grep -r -l "Outbox\|outbox" --include="*.java" 2>/dev/null | head -5

# SQL маркеры
grep -r -l "outbox" --include="*.sql" --include="*.yaml" --include="*.xml" 2>/dev/null | head -5

# Партиальный индекс на sent_at IS NULL — самый надёжный маркер
grep -r "sent_at IS NULL\|sent_at is null" --include="*.sql" --include="*.java" 2>/dev/null | head
```

Если нашёл — прочитай 1-2 файла, чтобы понять схему таблицы (поля, индексы) и **как именно** реализован publisher (poll-based, listener-based, debezium CDC).

`patterns.outbox: true|false`

## Идемпотентность

```bash
grep -r -l "Idempotency-Key\|IdempotencyKey\|idempotency_key" \
  --include="*.java" --include="*.sql" 2>/dev/null | head -5
```

Если нашёл — определи:
- Storage: отдельная таблица или Redis или другое
- TTL: `WHERE created_at < NOW() - INTERVAL '...'` в чистке
- Replay: что хранится — только статус или полный response

`patterns.idempotency: true|false`

## 3DS2 / платёжные потоки

```bash
grep -r -l "threeDS\|3ds\|ChallengeRequest\|ARes\|CRes\|EMV" \
  --include="*.java" 2>/dev/null | head -5
```

Если нашёл — проект работает с реальными платежами, в CLAUDE.md добавь раздел про PCI DSS требования (маскирование PAN, CVV не хранится, и т.п.).

`patterns.3ds2: true|false`

## Locking patterns

```bash
# JPA
grep -r "@Modifying\|@Lock\|LockModeType" --include="*.java" -l 2>/dev/null | head

# Native SQL locking
grep -r "FOR UPDATE\|SKIP LOCKED" --include="*.java" -l 2>/dev/null | head

# Java concurrency
grep -r "ReentrantLock\|StampedLock\|ReadWriteLock\|Semaphore" --include="*.java" -l 2>/dev/null | head

# Distributed locks
grep -r "ShedLock\|RedisLockRegistry\|Redisson" --include="*.java" -l 2>/dev/null | head
```

Зафиксируй какие используются — это влияет на скилл `scheduled-job` (нужен ли распределённый лок).

```json
"patterns": {
  "uses_jpa_locking": true,
  "uses_distributed_lock": "shedlock|redis|none"
}
```

## Circuit breakers / Resilience

```bash
grep -r "Resilience4j\|@CircuitBreaker\|@Retry\|@RateLimiter\|@Bulkhead" --include="*.java" -l 2>/dev/null | head
```

`patterns.circuit_breaker: true|false`

## OpenAPI-first

```bash
# OpenAPI спеки
find . -path "*/openapi/*" \( -name "*.yaml" -o -name "*.yml" \) 2>/dev/null | head

# Генератор в pom.xml
grep -r "openapi-generator-maven-plugin\|org.openapitools" $(find . -name "pom.xml" -not -path "*/target/*") | head

# Сгенерированные DTO
find . -path "*/generated-sources/openapi/*" -name "*.java" 2>/dev/null | head -3
```

Если есть и спеки, и плагин, и controllers `implements <Generated>Api` — `openapi_first: true`.
Если есть только аннотации `@RequestMapping` без спек — `openapi_first: false` и в CLAUDE.md этот паттерн пропускается.

## Logging / Observability

```bash
# Structured logging
grep -r "MDC\|@Slf4j\|LoggerFactory" --include="*.java" -l | head

# Correlation ID
grep -r "correlation\|x-request-id\|X-Request-Id\|traceId" --include="*.java" --include="*.yaml" -l | head

# Маскирование PII
grep -r "mask\|MaskingFilter\|@JsonProperty.*hidden" --include="*.java" -l | head

# Metrics (Micrometer)
grep -r "Micrometer\|MeterRegistry\|@Timed\|@Counted" --include="*.java" -l | head
```

```json
"observability": {
  "uses_mdc": true,
  "has_correlation_id": true,
  "has_pii_masking": true,
  "uses_micrometer": true
}
```

Если PII masking нет — это **высокий приоритет** для CLAUDE.md в fintech (отметь как gap).

## Что делать с найденным

В фазе генерации:
- `outbox: true` → предлагай скилл `outbox-event` в интервью (рекомендован)
- `idempotency: true` → предлагай `idempotency-handler` (рекомендован)
- `outbox: false` + Kafka используется → подними в интервью: "не нашёл Outbox, события идут напрямую — это намеренно?"
- `pii_masking: false` + 3DS2 → серьёзный ⚠️ в CLAUDE.md, обязательно подними в интервью

## Подводные камни

- **Маркеры могут быть в comments / dead code.** Если нашёл "Outbox" только в одном TODO-комментарии — паттерна по факту нет.
- **Частичная имплементация.** Outbox-таблица есть, но publisher закомментирован — функционально паттерна нет. Прочитай 2-3 файла прежде чем уверенно ставить `true`.
- **Generated code.** `target/generated-sources/` — сгенерированный код, его читать не нужно. Все grep исключают `target/`.
