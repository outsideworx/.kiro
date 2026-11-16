# Troubleshooting

Symptom → likely cause → fix, for failures that have actually happened or follow directly from a documented rule. Check the symptom first, then open the linked document for detail. All fixes that need a deploy are handed to the user to run (see `ways-of-working`); diagnose with read-only commands only (`docker service ps`, `docker service logs`, `docker logs`, `docker compose config`).

## Routing and networking

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| A site's domain returns `503 no available server` | The service has no `networks:` entry, so it only joins the implicit `sites_default` overlay, which Traefik is not on (happened to `tunde-divat`) | Put the service on `edge` (backend-free) or `outsideworx` (see `networking.md`, `new-site` skill) |
| Traefik routes but the site stays "unhealthy" | `/metrics` is missing or answers non-200 (static sites ship `metrics.txt`; the npm app has an Express route) | Add/restore `metrics.txt` (`up 1`) or the `/metrics` route; the health check path is `/metrics` on port 80 |
| Spring Boot cannot reach Postgres/Authelia, or hostname errors mention underscores | Java rejects `_` in hostnames (RFC 3986); the `services_*` VIP names do not work for Java | Use the dash aliases (`services-postgres`, `services-authelia`); see `networking.md` |
| Aliases (`services-postgres`, etc.) do not resolve | Aliases are task-scoped: they only resolve on the node running the aliased task | Keep `authelia`, `postgres` and `services` on the same node |
| Prometheus cannot scrape the Spring Boot actuator | Target uses `services_services` and Spring rejects the underscore Host header | Target must be `services-services:81` (alias) |
| `wg-easy` Prometheus target is down | The metrics endpoint is not enabled in the wg-easy admin UI or its password is not the `wg_metrics_token` value | Enable Prometheus metrics in the admin UI and set the password to the token printed by `deploy.sh --secrets` (see `wg-easy.md`) |
| A site target is down and `/metrics` is served as `text/html` | Prometheus 3.x rejects `text/html`; Express defaults string bodies to it | Return `text/plain; version=0.0.4` explicitly (see `monitoring.md`) |

## Auth and API

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| After login via Authelia the browser loops or lands on an error page | No Thymeleaf template/`ModelVisitor` for the user's email domain (`ErrorController` logs "Supressing Whitelabel Error Page" and redirects to Authelia) | Create `templates/clients/<domain-prefix>.html` and a `ModelVisitor` returning `clients/<domain-prefix>`; see `admin-portal` skill |
| `/api/**` calls are rejected | `X-Caller-Id`/`X-Auth-Token` pair matches no configured client: wrong `TOKEN` in the site's compose, missing `APP_CLIENTS_<CLIENT>_TOKEN` in services compose/`.env`, or a different token in the two `.env` files | Make both `.env` files carry the same token and check both compose files (see `auth.md`, `new-client` skill) |
| Browser blocks an API call (CORS) | Request origin is not the client's configured `origin`; calls from the `outsideworx.net/clients/<name>` mirror originate from `outsideworx.net`, not the official domain | Check `app.clients.<client>.origin` and `@CrossOrigin`; see the CORS note in `auth.md` |
| App fails at startup with a proxy/CGLIB error | A `final` class carries a proxy-based annotation (`@CacheEvict`, `@Cacheable`, `@Transactional`, `@Async`) | Remove `final` from that class (see `coding-conventions`) |
| The `tunde-divat` container exits right after start | The entrypoint runs under `set -e`: a failed `prisma migrate deploy`/seed stops it. Env is validated at import time, so a missing or short (<32 chars) `SESSION_SECRET` also crashes boot by design | Read `docker service logs sites_tunde-divat`; fix the env value or migration |

## Data and cache

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Thumbnail grid on a gallery site is empty | API error or empty response; the frontend degrades silently (`console.error`) | Check the browser console, then the services logs for the request id (`X-Request-Id`); see the site repo's `AGENTS.md` |
| Edits in the admin portal do not show on the site | Cached response: `/api/cache/**` carries `Cache-Control: public, max-age=86400`, plus the Caffeine cache if the POST path lacks `@CacheEvict(allEntries = true)` | Hard-reload to bypass the browser cache; confirm the controller evicts |
| Cached image files are missing or stale | `cache.py` polls every 60 s and only re-exports rows whose `hash` changed. A new image table without a hash trigger or `cache.py` entry never syncs | Add the `001-hash-trigger-*` changeset and the `cache.py` entry (see `liquibase` skill, `python-utils.md`) |
| `utils` task rejected with `invalid mount config ... bind source path does not exist` | Old `./utils:/etc/utils` bind mount where the Docker daemon does not share the repo path | Prod uses the `utils.tar` Swarm config instead; make sure `deploy.sh` ran from the current repo (see `services-deployment.md`) |

## CI and deploy

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| A push to a site repo did not go live | The site's `build.yaml` dispatch failed, or the `sites` `build` job failed before its final `docker service update --force` step | Check the Actions runs of the site repo, then of `sites`; see `github-actions.md` |
| A push to `sites/main` rebuilt images but nothing changed in production | The matrix `build-sites` job only pushes to GHCR; stack changes need the manual `deploy.yaml` (or `deploy.sh`) | Hand the deploy to the user |
| No verify/build ran for a commit | Message starts with `DOC:` (skips `services` verify, and the `sites` matrix build) | Intentional; use another prefix to build |
| `update-submodule` job fails with "nothing to commit" | The submodule pointer did not move; the step commits unconditionally | Harmless — nothing to deploy |
| `tunde-divat` is down for 30–60 s on each deploy | It runs Swarm defaults (`stop-first`, `pause`) on purpose, because the entrypoint writes to the SQLite volume on every boot | Expected; do not switch it to `start-first` |
| A new domain gets no TLS certificate | Let's Encrypt HTTP challenge needs the domain's DNS to point at the host | Fix DNS, then check Traefik logs |
| A WIP page redirects to a secret form | The `CLIENT_SECRET` cookie gate on `/clients/thegreen/` | Enter the secret (`APP_CLIENTS_THEGREEN_SECRET`); see `sites-wip.md` |
