# Stack Detection

Как определить версии ключевых библиотек проекта.

## Maven

```bash
# Все pom.xml кроме target
PXMLS=$(find . -name "pom.xml" -not -path "*/target/*" -not -path "*/.idea/*")

# Spring Boot version
grep -h "spring-boot" $PXMLS | grep -oP '<version>[^<]+' | head -1

# Java version
grep -h -E "(java.version|maven.compiler.target)" $PXMLS | head -3

# Ключевые библиотеки
grep -h -B1 -A1 -E "(kafka|liquibase|wiremock|rest-assured|assertj|openapi-generator|springdoc|testcontainers|mapstruct|lombok|resilience4j)" $PXMLS | \
  grep -E "(artifactId|version)" | sort -u
```

## Gradle

```bash
# Версии в build.gradle
grep -h -E "(springBoot|kafka|liquibase|wiremock|restAssured|assertj|openApiGenerator|testcontainers|mapstruct|lombok|resilience4j)" \
  build.gradle build.gradle.kts settings.gradle settings.gradle.kts 2>/dev/null

# Версии в gradle.properties
cat gradle.properties 2>/dev/null
```

## Что заполнить в context

```json
"stack": {
  "build_system": "maven",
  "java_version": "17",
  "spring_boot": "3.2.1",
  "kotlin": false
},
"libs": {
  "kafka_client": "3.6.0",
  "liquibase": "4.25.0",
  "wiremock": "3.5.4",
  "rest_assured": "5.4.0",
  "assertj": "3.24.2",
  "openapi_generator": "7.2.0",
  "testcontainers": "1.19.3",
  "mapstruct": "1.5.5.Final",
  "lombok": "1.18.30",
  "resilience4j": null
}
```

## Что делать если версии не нашёл

- **`spring_boot` не найден** → проект не Spring Boot, остановись и сообщи пользователю.
- **`wiremock` не найден** → не предлагай скилл `wiremock-integration-test` в интервью.
- **`openapi_generator` не найден** → не предлагай `openapi-first-endpoint`, спроси в интервью используют ли OpenAPI вообще.
- **`testcontainers` не найден** → отметь как TDD-blocker.

## Подводные камни

- В монорепо разные модули могут иметь разные версии одной библиотеки. Если так — выбери самую свежую или ту что в "ключевых" модулях. Уточни в интервью если расхождение значимое.
- В корневом pom.xml версии часто в `<dependencyManagement>`, не в `<dependencies>`.
- Spring Boot BOM может скрывать конкретные версии — для `kafka-clients`, `assertj` смотри `mvn dependency:tree -Dincludes=...`, если нужна точность.
