# GitLab Detection

Сбор данных для скилла `create-mr`.

## Project path и URL

```bash
# Из git remote
git remote get-url origin 2>/dev/null
# Примеры:
#   git@gitlab.example.com:fintech/bnpl-gateway.git
#   https://gitlab.example.com/fintech/bnpl-gateway.git

# Извлечь path: fintech/bnpl-gateway
git remote get-url origin 2>/dev/null | sed -E 's|.*[:/]([^/]+/[^/]+)\.git$|\1|'
```

`git.project_path: "fintech/bnpl-gateway"`
`git.host_url: "https://gitlab.example.com"`

## Default branch

```bash
# Из локального git
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@'

# Из CI
grep -E "^(default|target_branch|MR_TARGET)" .gitlab-ci.yml 2>/dev/null
grep -E "if.*branch ==" .gitlab-ci.yml 2>/dev/null | head
```

Обычно `develop` или `main`. Зафиксируй в `git.default_branch`.

## Branch naming convention

Прямого источника обычно нет, но косвенные признаки:

```bash
# Существующие ветки на origin
git branch -r 2>/dev/null | grep -v HEAD | head -30

# В .gitlab-ci.yml могут быть rules по branch pattern
grep -E "(branches:|^\s+- if:.*branch)" .gitlab-ci.yml 2>/dev/null
```

Типичные паттерны (определи по существующим веткам):
- `feature/BNPL-1234-description`
- `feat/refund-creation`
- `BNPL-1234-refund`

Если из веток не понятно — оставь TODO и спроси в интервью.

## MR templates

```bash
ls .gitlab/merge_request_templates/ 2>/dev/null
cat .gitlab/merge_request_templates/Default.md 2>/dev/null | head -50
```

Если есть — `git.mr_template_path` указывает на шаблон. Скилл `create-mr` будет его заполнять.

## CI обязательные джобы

```bash
# Какие stages есть
grep -E "^stages:|^\s+stage:" .gitlab-ci.yml 2>/dev/null | head -20

# Тест-джобы
grep -B1 -E "(test|verify|integration|sonar|lint)" .gitlab-ci.yml 2>/dev/null | head -30
```

Зафиксируй какие джобы должны быть зелёные перед мержем — это попадёт в `create-mr` Phase 3 как "обязательные локальные проверки".

## Commit message convention

Глянь последние коммиты:
```bash
git log --oneline -30 2>/dev/null
```

Определи паттерн:
- `feat(refund): add POST /refunds` → Conventional Commits
- `BNPL-1234: add refund endpoint` → Ticket-prefix
- `Add refund endpoint` → Свободный

## glab CLI доступность

Скилл `create-mr` будет использовать `glab` для создания MR (если установлен) или fallback на curl + GitLab API.

```bash
which glab 2>/dev/null
glab auth status 2>/dev/null
```

`git.glab_available: true|false`
`git.glab_authenticated: true|false`

## Что заполнить в context

```json
"git": {
  "host": "gitlab",
  "host_url": "https://gitlab.example.com",
  "project_path": "fintech/bnpl-gateway",
  "default_branch": "develop",
  "branch_naming": "feature/BNPL-NNNN-description",
  "commit_convention": "conventional|ticket-prefix|free",
  "has_mr_template": true,
  "mr_template_path": ".gitlab/merge_request_templates/Default.md",
  "ci_required_jobs": ["test", "integration-test", "sonar"],
  "glab_available": true,
  "glab_authenticated": false
}
```

## Что делать если что-то не нашёл

- **Нет git remote** → `create-mr` не будет работать, отметь в интервью.
- **Не GitLab** → этот набор скиллов не поддерживает GitHub/Bitbucket в `create-mr`. Скажи пользователю и предложи всё равно сгенерировать остальные скиллы (без `create-mr`).
- **Нет .gitlab-ci.yml** → отметь, в `create-mr` Phase 3 будет только локальный прогон тестов.
- **Нет MR template** → `create-mr` будет использовать дефолтный простой шаблон (что сделано / как тестировалось / blast radius).
- **glab нет** → `create-mr` будет использовать curl к GitLab API. В интервью уточни наличие `GITLAB_TOKEN` env variable.

## Подводные камни

- **Self-hosted GitLab.** URL может быть `https://gitlab.company.local`, а не `gitlab.com`. Не хардкодь `gitlab.com` нигде.
- **HTTPS vs SSH в remote.** Парсинг path должен работать с обоими форматами.
- **default_branch может отличаться от того, куда идут MR.** На многих проектах `main` — это релизная, а MR идут в `develop`. Уточни в интервью.
