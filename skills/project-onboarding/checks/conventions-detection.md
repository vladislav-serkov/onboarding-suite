# Conventions Detection

Как найти naming patterns проекта (модули, пакеты, SQL, Kafka, HTTP).

## Корневой Java пакет

```bash
# Топ-уровневые пакеты в каждом модуле
for src in $(find . -name "src" -type d -not -path "*/target/*"); do
  find "$src/main/java" -mindepth 4 -maxdepth 5 -type d 2>/dev/null | head -3
done
```

Извлеки общий префикс. Пример: все пути ведут в `src/main/java/ru/mts/bnpl/...` → `package_root: "ru.mts.bnpl"`.

## Naming модулей

```bash
# Из корневого pom.xml
grep -E "<module>" pom.xml 2>/dev/null

# Или физически
ls -d */ 2>/dev/null | grep -E "^(bnpl|flp|payment|gateway)-"
```

Определи паттерн: `bnpl-*`, `flp-*`, `<company>-<service>-<layer>`, или mixed.

## SQL schema prefix

```bash
# Liquibase changelog'и
CHANGELOGS=$(find . -path "*/db/changelog/*" \( -name "*.sql" -o -name "*.yaml" -o -name "*.xml" \) 2>/dev/null | head -10)

# Префикс в SQL
grep -h -oP "(?<=schemaName:\s)[a-z_]+" $CHANGELOGS | sort -u
grep -h -oP "(?<=CREATE TABLE\s)[a-z_]+\." $CHANGELOGS | sort -u

# Префикс в @Table аннотациях
grep -r "@Table" --include="*.java" -h | grep -oP 'schema\s*=\s*"[^"]+' | sort -u
```

Если нашёл `flp_baas_facade.payment` — `sql_schema: "flp_baas_facade"`.

Если нет префикса нигде → `sql_schema: "public"` (дефолт PostgreSQL) и **отметь как анти-паттерн** для CLAUDE.md.

## Liquibase changeset naming

```bash
# Несколько последних changeset id
grep -r "id:" $CHANGELOGS 2>/dev/null | head -20
```

Определи формат:
- `20260508-01-add-refund-table` → `YYYYMMDD-NN-description`
- `0001-create-tables` → `NNNN-description`
- `add-refund-table-v2` → `description-vN` (нестрогий)

## HTTP path template

```bash
# Из @RequestMapping / @GetMapping и т.п.
grep -r "@RequestMapping\|@GetMapping\|@PostMapping\|@PutMapping\|@DeleteMapping\|@PatchMapping" \
  --include="*.java" -h | \
  grep -oP '"/[^"]+"' | sort -u | head -30

# Из OpenAPI спек
find . -path "*/openapi/*" \( -name "*.yaml" -o -name "*.yml" \) -exec grep -h -oE "^\s+/[a-z][a-z0-9/{}_-]+:" {} \; | sort -u | head -30
```

Найди общий префикс. Примеры:
- Все пути начинаются с `/api/v1/...` → template `/api/v1/{domain}`
- Mixed `/api/v1` и `/api/v2` → template `/api/{version}/{domain}`, отметь что есть версионирование
- `/bnpl/v2/...` → template `/bnpl/{version}/{domain}`

## Kafka topic naming

```bash
# В Java коде
grep -r "topic\s*=\|@KafkaListener\|KafkaTemplate" --include="*.java" -h | \
  grep -oP '"[a-z][a-z0-9.\-]+\.(queue|topic|out|in|dlq)"' | sort -u

# В application.yaml
grep -E "topic" $(find . -name "application*.yaml" -o -name "application*.yml" -o -name "application*.properties") 2>/dev/null | \
  grep -oE "[a-z][a-z0-9.\-]+\.(queue|topic|out|in|dlq)" | sort -u
```

Разбери паттерн на сегменты. Пример `pay-later.gateway.payment.created.out.queue`:
- `pay-later` — bounded context
- `gateway` — service
- `payment` — domain
- `created` — action / event type
- `out` — direction (out = публикуемся, in = слушаем)
- `queue` — type suffix

Template: `{context}.{service}.{domain}.{action}.{direction}.queue`.

## Что заполнить в context

```json
"conventions": {
  "package_root": "ru.mts.bnpl",
  "module_naming": "bnpl-*",
  "sql_schema": "flp_baas_facade",
  "changeset_naming": "YYYYMMDD-NN-description",
  "http_path_template": "/api/v1/{domain}",
  "kafka_topic_template": "{context}.{service}.{domain}.{action}.{direction}.queue",
  "kafka_topic_example": "pay-later.gateway.payment.created.out.queue"
}
```

## Подводные камни

- В большом проекте могут быть **исключения** из паттерна. Например legacy-модуль `old-payment-system` без префикса. Зафиксируй это в context и подними в интервью как "off-limits для агента".
- HTTP path template может различаться между modules: gateway-уровень `/api/v1/`, internal-уровень `/internal/v1/`. Это разные шаблоны — фиксируй оба.
- В Kafka топиках направление (`out`/`in`) важно: `out` для producer'ов, `in` для consumer'ов. Если не различают — это анти-паттерн, зафиксируй.
