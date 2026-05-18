# Сервис коротких ссылок

Превращает длинные ссылки в `https://to.profcomff.com/<alias>`. OIDC-аутентификация
через `auth.profcomff.com`, soft-delete ссылок с сохранением истории переходов,
бар-чарты переходов за час/день/всё время.

- Production: <https://to.profcomff.com/>
- Testing: <https://to.test.profcomff.com/>

## Стек

- **API:** Python 3.13 + FastAPI + SQLAlchemy/alembic. Внешний Postgres (по `DB_DSN`).
- **Web:** Vite + React 19 + TypeScript + Tailwind 4 SPA (PWA).
- **Auth:** HS256 access-JWT (15 мин, in-memory у фронта) + HttpOnly refresh-cookie
  (sliding 30 дней). OIDC-флоу через `/api/v2/oidc/redirect` + `/api/v2/oidc/callback`.
- **Инфра:** Docker Compose поверх существующей external-сети `web`. Reverse-proxy
  на хосте.

Исходники сервиса закрытые; в этом репо живёт только деплой-обвязка.

## Что в репо

```
deploy/compose.yaml             prod docker-compose (API + UI, БД внешняя)
.github/workflows/deploy.yml    GitHub Actions: testing + production
```

Образы тянутся из `git.dyakov.space/dyakov-space/to/{control-panel-service,web}`
(приватный Gitea-registry, доступ через `secrets.DYAKOVSPACE_CI_TOKEN`).

## Деплой

| Триггер | Окружение | Runner | Тег образов | Project name |
|---------|-----------|--------|-------------|--------------|
| push в `main` | Testing | `[self-hosted, Linux, testing]` | `dev-latest` | `com_profcomff_redirect_test` |
| push тега `v*` | Production | `[self-hosted, Linux, production]` | `latest` | `com_profcomff_redirect` |

Шаги одинаковые: `docker compose pull` → `docker compose run --rm
control-panel-service ... alembic upgrade head` → `docker compose up -d
--remove-orphans`. Обе среды мигрируют свою БД отдельно.

Имена контейнеров (`com_profcomff_api_redirect[_test]`,
`com_profcomff_ui_redirect[_test]`) сохранены ради существующих reverse-proxy
конфигов. Дополнительно сервисы получают network-aliases `redirector-api` и
`redirector-www` на сети `web`.

## Требования к окружению на раннере

- `docker` + плагин `docker compose` v2.
- Существующая external network `web` (`docker network create web`).
- Доступ к Postgres-инстансу, который указан в `DB_DSN` для этой среды.

## Переменные в GitHub Environments

Задаются вручную в Settings → Environments → Testing/Production.

**Secrets** (одинаковый набор в обоих окружениях, разные значения):

| Имя | Назначение |
|-----|-----------|
| `DYAKOVSPACE_CI_TOKEN` | Read-токен к `git.dyakov.space` для `docker login` |
| `DB_DSN` | `postgresql://...` к внешней БД (своя на каждом окружении) |
| `JWT_SECRET_KEY` | HS256-ключ подписи наших access-токенов (`openssl rand -hex 32`) |
| `OIDC_CLIENT_SECRET` | OIDC client secret |
| `OIDC_TRUSTED_TOKEN` | Опционально. Bearer-bypass для dev. В проде оставлять пустым |

**Vars:**

| Имя | Пример (production) |
|-----|--------------------|
| `BASE_URL` | `https://to.profcomff.com` |
| `OIDC_CONFIGURATION_URI` | `https://auth.profcomff.com/application/o/redirector/.well-known/openid-configuration` |
| `OIDC_CLIENT_ID` | `redirector` |
| `OIDC_ADMIN_CLAIM` | `groups` |
| `OIDC_ADMIN_CLAIM_VALUE` | `redirector-admin` |
| `ALLOWED_DOMAINS` | `["https://to.profcomff.com"]` |

## Ручной запуск на раннере

```bash
cd /path/to/checkout
PROJECT_VERSION=latest CONTAINER_SUFFIX= \
DB_DSN=... BASE_URL=... JWT_SECRET_KEY=... \
OIDC_CONFIGURATION_URI=... OIDC_CLIENT_ID=... OIDC_CLIENT_SECRET=... \
OIDC_ADMIN_CLAIM=groups OIDC_ADMIN_CLAIM_VALUE='Redirector Admin' \
ALLOWED_DOMAINS='["https://to.profcomff.com"]' \
docker compose --project-directory deploy --project-name com_profcomff_redirect pull

docker compose --project-directory deploy --project-name com_profcomff_redirect \
  run --rm control-panel-service \
  uv run --active --no-sync --directory=backend/control-panel-service alembic upgrade head

docker compose --project-directory deploy --project-name com_profcomff_redirect \
  up --detach --remove-orphans
```
