# CLAUDE.md — onboarding-suite

Этот репозиторий — **Claude Code plugin marketplace**. Контент из `plugins/onboarding-suite/` устанавливается у пользователей через `/plugin install` и обновляется командой `/onboarding-suite:update` (см. `plugins/onboarding-suite/commands/update.md`).

## Контракт версионирования (enforced)

Любые правки внутри `plugins/<name>/` (SKILL.md, phases, commands, templates, checks, agents) **обязаны** сопровождаться bump'ом `version` в `plugins/<name>/.claude-plugin/plugin.json`:

- **patch** (`0.2.1`) — правки текста, фиксы инструкций, опечатки.
- **minor** (`0.3.0`) — новые commands/skills/templates, новые опции.
- **major** (`1.0.0`) — несовместимые изменения формата или удаление публичных команд.

Правки вне `plugins/` (этот `CLAUDE.md`, `README.md`, `.githooks/`, корневой `.claude-plugin/marketplace.json`-метадата) — **без bump'а** версии плагина.

Правило enforced через `.githooks/pre-push`: попытка push'а с изменениями в `plugins/<name>/` без bump'а будет отклонена.

### Setup hook (один раз после clone)

```bash
git config core.hooksPath .githooks
```

## Правила работы для Claude

1. **Проактивно предлагай push после завершённой правки.** После любой логически законченной правки внутри `plugins/<name>/` сразу предложи юзеру: bump version + commit + push **одной операцией**. Не жди вопроса «а закоммитим?».
2. **Не пушь WIP.** Если правка полусырая или часть большей задачи — спроси, готова ли версия к релизу, прежде чем предлагать push.
3. **Выбор bump'а** делай сам по правилам выше, но проговори юзеру (например: «правки только в SKILL.md → patch, `0.2.0 → 0.2.1`»).
4. **Перед commit'ом** показывай `git status` и `git diff --stat`, чтобы юзер видел scope.
5. **Сообщение коммита** — английский, императив, одно предложение в subject + body с обоснованием «зачем». Без emoji.
6. **После push'а** напомни юзеру: `/onboarding-suite:update` → `/reload-plugins` чтобы получить свежую версию у себя.

## Структура репо

```
.claude-plugin/marketplace.json          # описание маркетплейса
.githooks/pre-push                       # enforces version bump
plugins/
  onboarding-suite/
    .claude-plugin/plugin.json           # манифест + version
    commands/                            # slash-команды плагина
      update.md                          # /onboarding-suite:update
    skills/
      project-onboarding/                # основной скилл
        SKILL.md
        phases/                          # 01-discovery, 02-interview, 03-generation
        checks/
        templates/
```
