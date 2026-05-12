# HANDOFF

## Current status

T1 deploy-layer preparation for self-hosted `things-cloud-mcp` on Hetzner.

This repo is a fork of `wbopan/things-cloud-mcp`. The deploy target is VPS `andrei@178.105.40.39`, path `~/apps/things-cloud-mcp/`, container `things_cloud_mcp`, behind Caddy on `things.myfrutilla.com`.

## Done

- Added `AGENTS.md` with repo purpose, stack, deployment contract, HTTP endpoint, auth boundary, secret handling, handoff protocol, and upstream sync notes.
- Added `docker-compose.yml` for Hetzner Docker deployment:
  - build from local Dockerfile;
  - `restart: unless-stopped`;
  - `user: "1000:1000"`;
  - named volume `things_data:/data`;
  - external Docker network `web`;
  - exposed container port `8788`.
- Added `.env.example` without real secrets.
- Updated `.gitignore` for local env and local-only files.

No Go code was changed.

## Left for Claude / VPS phase

- Create/copy real `.env` on VPS with `chmod 600`.
- Deploy to `~/apps/things-cloud-mcp/` on Hetzner.
- Add Caddy block:

```caddyfile
things.myfrutilla.com {
  reverse_proxy things_cloud_mcp:8788
}
```

- Run Docker compose build/start.
- Smoke test `/mcp` with Basic auth and verify tool names:
  - `things_create_task`;
  - `things_find_areas`;
  - `things_find_tags`;
  - `things_overview`.
- Add monitoring cron/alert on VPS for `/mcp` 5xx and repeated Basic auth 401s.

## Notes / caveats

- Current Dockerfile does not set `USER`; compose-level `user: "1000:1000"` has no appuser conflict.
- Current Dockerfile has `EXPOSE 8080` and `ENV PORT=8080`; compose overrides runtime `PORT` to `8788`.
- Current Go code reads `PORT`, `DATA_DIR`, `JWT_SECRET`, `PROXY_URLS`, and `THINGS_DEBUG`.
- `OAUTH_ENABLED` is included for the T1/T6 deployment boundary, but current Go code does not read it. OAuth routes are currently registered by the upstream server regardless of this env var. T1 keeps cloud-client use disabled operationally; do not wire Claude.ai/Perplexity until T6.
