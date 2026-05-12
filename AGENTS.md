# things-cloud-mcp

## Назначение

`things-cloud-mcp` — self-hosted MCP server для Things Cloud (Cultured Code). Это fork `wbopan/things-cloud-mcp` под MIT license.

Продакшен деплоится на Hetzner VPS `andrei@178.105.40.39` как Docker-контейнер `things_cloud_mcp` за Caddy на `things.myfrutilla.com`.

## Стек

- **Runtime**: Go 1.25.0.
- **MCP SDK**: `github.com/mark3labs/mcp-go`.
- **Storage**: `modernc.org/sqlite` для OAuth state, refresh tokens и JWT secret.
- **Things Cloud**: `github.com/arthursoares/things-cloud-sdk`, подключен как submodule/local module через `replace` directive в `go.mod`.
- **HTTP transport**: Streamable HTTP MCP.

## Деплой

- VPS path: `~/apps/things-cloud-mcp/`.
- Docker Compose: `docker-compose.yml`.
- Container name: `things_cloud_mcp`.
- Local/container port: `8788`.
- Persistent state: named volume `things_data:/data`.
- Docker network: external `web`, общий reverse-proxy network для Caddy.
- Restart policy: `unless-stopped`.
- Runtime user: `1000:1000`.

## HTTP endpoint

- MCP endpoint: `https://things.myfrutilla.com/mcp`.
- Auth:
  - `Authorization: Basic <base64(email:password)>` для trusted clients.
  - `Authorization: Bearer <token>` для orchestration/cloud-client flows после включения OAuth use case.

Базовая T1-проверка должна смотреть на имена tools, а не на общее число tools. Обязательный минимум: `things_create_task`, `things_find_areas`, `things_find_tags`, `things_overview`.

## Environment

```env
PORT=8788
DATA_DIR=/data
JWT_SECRET=<generated>
OAUTH_ENABLED=false
```

`PORT` и `DATA_DIR` читаются текущим Go-кодом. `JWT_SECRET` читается текущим Go-кодом; если он не задан, сервер генерирует secret и сохраняет его в SQLite в `DATA_DIR/oauth.db`.

`OAUTH_ENABLED=false` — deployment policy flag для T1/T6 boundary. На текущем upstream-состоянии код не читает эту переменную и OAuth routes регистрируются всегда; в T1 cloud-client access не подключается операционно.

## Multi-client boundary

**Basic auth** — для доверенных клиентов:

- `companion-bot`;
- Claude Code;
- Codex CLI;
- другие локальные/серверные агенты под нашим контролем.

Каждый trusted client хранит `THINGS_EMAIL` и `THINGS_PASSWORD` в своём `.env` и пробрасывает их как Basic header. Things Cloud сама валидирует эти credentials.

**OAuth 2.1** — для cloud-clients:

- Claude.ai;
- Perplexity;
- другие внешние клиенты, которым нельзя отдавать Things credentials.

OAuth не должен отдавать cloud-clients Things email/password. Включать и подключать его только в T6, когда появится конкретный cloud-client.

## Secret handling

`JWT_SECRET` живёт в persistent state named volume `things_data` через `/data/oauth.db`. При первом старте можно задать `JWT_SECRET` в `.env` или оставить пустым, чтобы сервер сгенерировал его и сохранил в DB. При стабильной эксплуатации secret не регенерируется.

`THINGS_EMAIL` и `THINGS_PASSWORD` не живут в этом репо и не передаются в контейнер server-wide. Они хранятся в `.env` конкретных trusted clients, например `companion-bot/.env`, с `chmod 600`.

Rotation procedure для Things credentials:

1. Сменить Things Cloud password в Cultured Code.
2. Обновить `.env` всех trusted clients, которые используют Basic auth.
3. Перезапустить эти clients.
4. Проверить `things_overview` или другой read-only tool.

`JWT_SECRET` ротировать только при компрометации сервера:

1. Остановить контейнер `things_cloud_mcp`.
2. Обновить persisted secret в `things_data` / `/data/oauth.db` или удалить старый persisted secret для regeneration.
3. Запустить контейнер заново.
4. Re-auth all OAuth/Bearer clients.

## HANDOFF protocol

Для передачи контекста между агентами:

1. Перед изменениями читать `HANDOFF.md`.
2. После завершения существенного шага обновлять `HANDOFF.md`: что сделано, что проверено, что осталось, какие риски.
3. Коммитить `HANDOFF.md` вместе с кодом/деплой-слоем.
4. Второй агент начинает с `git pull`, читает `AGENTS.md` и `HANDOFF.md`, затем продолжает работу.

GitHub — основной handoff surface между Claude и Codex по конкретным задачам. Runtime source of truth остаётся на VPS и в Docker volumes, а не в markdown-документации.

## Upstream sync

Этот репозиторий является fork upstream `wbopan/things-cloud-mcp`.

Периодически, вручную примерно раз в месяц:

```bash
git fetch upstream
git merge upstream/main
```

Цель sync: security fixes, изменения Things Cloud API compatibility, новые tools и OAuth fixes. После merge обязательно проверить deploy layer, `go test ./...`, Docker build и smoke test `/mcp`.
