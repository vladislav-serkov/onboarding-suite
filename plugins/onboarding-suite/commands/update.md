---
description: Обновить плагин onboarding-suite до последней версии из GitHub (git pull + rsync в plugin cache, без /plugin uninstall+install)
---

# /onboarding-suite:update

Обновить установленный плагин `onboarding-suite` до последней версии из ветки `main` репозитория `vladislav-serkov/onboarding-suite`.

## Что сделать

Выполни шаги по порядку. После каждого Bash — проверь exit code, при ошибке остановись и покажи юзеру что упало.

### 1. Найти clone маркетплейса и подтянуть main

```bash
MP_DIR=~/.claude/plugins/marketplaces/onboarding-suite
if [ ! -d "$MP_DIR/.git" ]; then
  echo "ERROR: $MP_DIR не git-репо. Маркетплейс не подключён или повреждён."
  echo "Подключи: /plugin marketplace add vladislav-serkov/onboarding-suite"
  exit 1
fi
git -C "$MP_DIR" fetch origin main
BEFORE=$(git -C "$MP_DIR" rev-parse HEAD)
git -C "$MP_DIR" reset --hard origin/main
AFTER=$(git -C "$MP_DIR" rev-parse HEAD)
echo "Marketplace: $BEFORE -> $AFTER"
```

### 2. Найти cache плагина

```bash
CACHE_BASE=~/.claude/plugins/cache/onboarding-suite/onboarding-suite
CACHE_DIR=$(ls -dt "$CACHE_BASE"/*/ 2>/dev/null | head -1)
if [ -z "$CACHE_DIR" ]; then
  echo "ERROR: cache плагина не найден. Плагин не установлен."
  echo "Установи: /plugin install onboarding-suite@onboarding-suite"
  exit 1
fi
echo "Cache: $CACHE_DIR"
```

### 3. Синхронизировать свежий контент в cache

```bash
SRC="$MP_DIR/plugins/onboarding-suite/"
rsync -a --delete "$SRC" "$CACHE_DIR"
echo "Synced $SRC -> $CACHE_DIR"
```

### 4. Показать changelog (коммиты между BEFORE и AFTER)

```bash
if [ "$BEFORE" != "$AFTER" ]; then
  echo "--- Changelog ---"
  git -C "$MP_DIR" log --oneline "$BEFORE..$AFTER"
else
  echo "Already up to date."
fi
```

### 5. Сообщить пользователю

После успешного rsync:
- Покажи новую версию из `cat "$CACHE_DIR/.claude-plugin/plugin.json" | python3 -c 'import json,sys; print(json.load(sys.stdin)["version"])'`.
- **Скажи юзеру перезапустить Claude Code** — обновлённые SKILL.md / commands / phases загрузятся только при следующей сессии.

## Если что-то пошло не так

- Маркетплейс не подключён → дай команду `/plugin marketplace add vladislav-serkov/onboarding-suite`.
- Плагин не установлен → дай команду `/plugin install onboarding-suite@onboarding-suite`.
- Конфликт в `git reset --hard` (не должно быть, но если cache_dir вручную правили) → не правь, репорти юзеру.
