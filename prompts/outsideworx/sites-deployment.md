# Sites Deployment

## Overview

The sites stack serves multiple websites, each built into its own container and routed by domain. Most sites are **static** (Apache httpd serving HTML/CSS/JS) and proxy API calls back to the services stack. Some sites are **dynamic npm apps** (Node/Express + Vite SPA with their own SQLite database) built from a separate Dockerfile.

## Architecture

```
GitHub (site repo) → repository_dispatch → sites repo CI → Docker build → GHCR
                                                                            ↓
Server: docker stack deploy → pulls images → Traefik routes by domain
```

Each site is a separate GitHub repository. Static sites contain HTML/CSS/JS (often Hugo-generated); the npm app is a full npm-workspaces monorepo. The `sites` repo provides the shared Dockerfiles and deployment config.

## Build Types (Dockerfile Selection)

The `sites` repo ships two families of Dockerfile, selected per site by a build **type**:

| Type | Dockerfile (prod / test) | Runtime base | Serves | Sites |
|------|--------------------------|--------------|--------|-------|
| (default, no type) | `Dockerfile` / `Dockerfile.test` | `httpd` (Apache) | Static content, proxies `/api/` | all static sites |
| `npm` | `Dockerfile.npm` / `Dockerfile.npm.test` | `httpd:2.4` + Node.js (builder: `node:22-alpine`) | Its own Express API + built SPA | `tunde-divat` |

The build type is passed through CI as `client_payload.type` (dispatch) or a `type` field in the build matrix (push). The workflow resolves the Dockerfile name dynamically: no type → `Dockerfile`, `npm` → `Dockerfile.npm` (see the `github-actions` prompt). Both types still take the `NAME` build arg to select which repo to clone.

## Dockerfile — Static Sites (Multi-Site, Single Template)

The same `Dockerfile` builds all static sites. The `NAME` build arg determines which site repo to clone.

### Build Stages

1. **fetcher** (`bitnami/git`) — Clones `https://github.com/outsideworx/${NAME}.git` with depth 1, initializes submodules
2. **runtime** (`httpd`) — Copies site content, configures Apache

### Apache Configuration

Modules enabled: `headers`, `lua`, `negotiation`, `proxy`, `proxy_http`, `ratelimit`, `remoteip`, `reqtimeout`, `unique_id`

#### Proxy to Services API

```apache
ProxyPass        "/api/"  "http://services_services/api/"
ProxyPassReverse "/api/"  "http://services_services/api/"
```

The proxy target is the Swarm service name (`services_services` = stack `services`, service `services`).

#### Request Headers Injected

```apache
RequestHeader set X-Auth-Token "${TOKEN}"
RequestHeader set X-Caller-Id "${NAME}"
RequestHeader set X-Request-Id "%{UNIQUE_ID}e"
```

`TOKEN` and `NAME` are set at container startup from environment variables.

#### Security Headers

- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `Referrer-Policy: strict-origin-when-cross-origin`
- `Content-Security-Policy` (restrictive: no inline scripts except `unsafe-inline`, no default-src)

#### Rate Limiting & Timeouts

- Output rate limit: 1536 KB/s (prod), 640 KB/s (test)
- Request read timeout: header 2-5s (MinRate 2048), body 5-30s (MinRate 4096)
- MPM event: 2 start servers, 16 min spare threads, 64 threads/child, 128 max workers (prod)

#### URL Blocking

```apache
RedirectMatch 403 /\.
RedirectMatch 403 \.(bak|conf|config|env|ini|json|key|log|properties|php|pub|py|sh|ts|yaml|yml|zip)/?$
RedirectMatch 403 ^(?!/(metrics|robots)\.txt$).*\.txt/?$
RedirectMatch 403 ^(?!/(sitemap)\.xml$).*\.xml/?$
```

#### Convenience Redirects

```apache
RedirectMatch 301 ^/grafana/?$  https://services.outsideworx.net/grafana
RedirectMatch 301 ^/login/?$    https://services.outsideworx.net
RedirectMatch 301 ^/ntfy/?$     https://services.outsideworx.net/ntfy
```

#### Logging

```
ErrorLogFormat "ERROR %P --- ip=%a requestId=%{UNIQUE_ID}e: %M"
LogFormat "INFO %P --- ip=%a requestId=%{UNIQUE_ID}e: %r %>s" log_format
```

`/metrics` requests are excluded from access log.

#### IP Blacklist

- Prod: `blacklist.conf` (large, real IPs)
- Test: `blacklist-test.conf` (minimal placeholder)

#### Content Negotiation

`Options +MultiViews` on htdocs root and `htdocs/clients/` (serves extensionless URLs).

### Entrypoint

The `CMD` runs a shell script that writes the `TOKEN` env var into an Apache config file at startup, then launches `httpd-foreground`.

## Dockerfile — npm Apps

`Dockerfile.npm` (prod) and `Dockerfile.npm.test` (test) build a dynamic npm app (currently `tunde-divat`). The two are nearly identical — the **only** difference is `NODE_ENV` (`production` in prod, `development` in test). The `NAME` build arg still selects the repo to clone, so the template stays multi-site-capable for any npm app.

### Build Stages

1. **fetcher** (`bitnami/git`) — Clones `https://github.com/outsideworx/${NAME}.git` (depth 1)
2. **builder** (`node:22-alpine`) — `npm ci`, then `npm run build` with `VITE_API_URL=""` (the SPA calls the API on its own origin under `/api/`)
3. **runtime** (`httpd:2.4` + `apk add --no-cache nodejs`) — Copies the built workspace to `/app`, sets `WEB_DIST_DIR=/app/apps/web/dist`, `NODE_ENV` (prod/test), and `API_PORT=80` as Dockerfile ENV, and bakes an entrypoint script at build time

The runtime image is Apache httpd with Node.js added on top — **not** a plain Node image. Both processes run in the container (see Entrypoint).

### Entrypoint

The entrypoint is written into the image at build time via `printf ... > /app/entrypoint.sh` (not generated at container startup). It runs, from `/app/apps/api`:

```sh
npx prisma migrate deploy   # apply SQLite migrations
npm run seed                # seed admin user + invite code
node dist/server.js &       # start Express on API_PORT=80 (backgrounded)
httpd-foreground            # run Apache in the foreground as PID 1
```

So the Express server is started in the background and Apache httpd runs in the foreground. Both share port 80 inside the container; Traefik routes to the Express server.

### Runtime Characteristics

- **Port**: Express listens on `API_PORT=80` (baked as a Dockerfile ENV) and serves both the API (`/api/`) and the built SPA (`WEB_DIST_DIR`). Apache httpd also runs on 80 in the same container.
- **Apache present but minimal**: unlike the static sites, there is no `/api/` proxy to services, no `TOKEN`/`X-Auth-Token` injection, no blacklist, and no MPM/rate-limit config — all request handling for the app lives inside the Express server.
- **No `/api/` proxy to services**: the app is self-contained; it does not call the Spring Boot backend
- **Persistence**: a SQLite DB file and uploads directory live on a named volume mounted at `/data` (`DATABASE_URL=file:/data/tunde-divat.db`, `UPLOAD_DIR=/data/uploads`)
- **Health check**: `wget --spider http://localhost/metrics` (compose) and Traefik healthcheck path `/metrics`, both on port 80
- **`/metrics` endpoint**: an Express route that returns the string `up 1`. It is used **only** by the Docker/Traefik health check — tunde-divat is **not** a Prometheus scrape target (see `monitoring.md`)

See the `tunde-divat` repo's `AGENTS.md` for the app's internal architecture (Express + React + Prisma, AI image generation, auth).

## deploy.sh

Simple deployment — no Swarm init needed (uses the network created by services).

### Flow

1. Creates `/home/outsideworx/sites/` (if missing)
2. Copies `.env`, `blacklist.conf`, `compose.yaml`
3. Sources `.env`
4. `docker compose pull`
5. `docker stack deploy -c compose.yaml sites --detach=false --resolve-image=always`
6. Force-updates all services

### Update Strategy

Every service in `compose.yaml` shares a YAML-anchored `update_config` (`x-update-config`):

```yaml
update_config:
  failure_action: rollback
  order: start-first
```

- **`order: start-first`** — Swarm starts the new task and waits for it to become healthy before stopping the old one, giving zero-downtime rolling updates (the health checks gate the cutover).
- **`failure_action: rollback`** — if the new task fails to converge, Swarm automatically rolls back to the previous task instead of leaving the service down.


### Prerequisites

- Swarm already initialized (by services deploy)
- `outsideworx` overlay network exists
- `.env` file present

## .env File (sites)

| Variable | Used by | Purpose |
|----------|---------|---------|
| `APP_CLIENTS_CIAFO_TOKEN` | come-in-and-find-out | API auth token (injected as `TOKEN`) |
| `APP_CLIENTS_PEEPS_TOKEN` | gaiapeeps | API auth token (injected as `TOKEN`) |
| `APP_CLIENTS_SOUP_TOKEN` | soupart | API auth token (injected as `TOKEN`) |
| `APP_CLIENTS_THEGREEN_SECRET` | outsideworx | Client secret for cookie-based access control (injected as `CLIENT_SECRET`) |
| `APP_CLIENTS_WORX_TOKEN` | outsideworx | API auth token (injected as `TOKEN`) |
| `TUNDE_DIVAT_AI_PROVIDER` | tunde-divat | AI image provider (`openai` or `mock`) → `AI_PROVIDER` |
| `TUNDE_DIVAT_OPENAI_API_KEY` | tunde-divat | OpenAI API key → `OPENAI_API_KEY` (only used when provider is `openai`) |
| `TUNDE_DIVAT_SEED_ADMIN_PASSWORD` | tunde-divat | Seeded admin password → `SEED_ADMIN_PASSWORD` |
| `TUNDE_DIVAT_SEED_ADMIN_USERNAME` | tunde-divat | Seeded admin username → `SEED_ADMIN_USERNAME` |
| `TUNDE_DIVAT_SEED_INVITE_CODE` | tunde-divat | Registration invite code → `SEED_INVITE_CODE` |
| `TUNDE_DIVAT_SESSION_SECRET` | tunde-divat | JWT/session signing secret (≥32 chars) → `SESSION_SECRET` |

Only static sites that call the API need a `TOKEN`. Static sites without API calls (duckumbrella, igli, soupkitchen) have no `TOKEN` environment variable. `outsideworx` has its own `TOKEN` (`APP_CLIENTS_WORX_TOKEN`, caller `outsideworx`) — in addition to its `CLIENT_SECRET` used to gate the `thegreen` submodule. It actively uses this token: the `/clients/<name>` mirror instances of come-in-and-find-out, gaiapeeps, and soupart are served from the `outsideworx` container, and their frontend JS makes `/api/` calls that the `outsideworx` Apache proxies to services, injecting `X-Caller-Id: outsideworx` and the WORX token. The `tunde-divat` npm app uses none of the `TOKEN`/`CLIENT_SECRET` mechanism — it has its own `TUNDE_DIVAT_*` variables (mapped to the Express app's env in compose). `DATABASE_URL`, `UPLOAD_DIR`, and `CORS_ORIGIN` are set as literals in `compose.yaml`, not via `.env`.

## Prod vs Test

### Static Sites

| Aspect | Prod (Dockerfile) | Test (Dockerfile.test) |
|--------|-------------------|------------------------|
| Blacklist | `blacklist.conf` (full) | `blacklist-test.conf` (minimal) |
| Proxy target | `http://services_services/api/` | `http://host.docker.internal:8080/api/` |
| Token injection | `TOKEN` env var at runtime | Hardcoded `"test"` |
| Token config | Via entrypoint script (`httpd-token.conf`) | Not used (hardcoded in proxy conf) |
| Rate limit | 1536 KB/s | 640 KB/s |
| MPM workers | 128 max | 4 max |
| Redirects | `https://services.outsideworx.net/...` | `http://localhost:8080/...` |
| Compose | `compose.yaml` (pulls from GHCR) | `compose-test.yaml` (builds locally) |
| Network | External overlay `outsideworx` | External `services_default` |
| Domains | Real domains | `*.localhost` |
| Health check interval | 1m | 5s |

### npm App (`tunde-divat`)

| Aspect | Prod (Dockerfile.npm) | Test (Dockerfile.npm.test) |
|--------|-----------------------|----------------------------|
| Image source | `compose.yaml` (pulls from GHCR) | `compose-test.yaml` (builds locally) |
| `NODE_ENV` | `production` (Dockerfile ENV) | `development` (Dockerfile ENV) |
| `AI_PROVIDER` | From `.env` (`TUNDE_DIVAT_AI_PROVIDER`) | `openai` (hardcoded) |
| `OPENAI_API_KEY` | From `.env` (`TUNDE_DIVAT_OPENAI_API_KEY`) | `""` (empty) |
| `CORS_ORIGIN` | `https://tundedivat.com` | `https://tunde-divat.localhost` |
| Seed vars | From `.env` (`TUNDE_DIVAT_SEED_*`) | Hardcoded test values |
| `SESSION_SECRET` | From `.env` (`TUNDE_DIVAT_SESSION_SECRET`) | Hardcoded 32+ char test string |
| Volume | Named volume `tunde-divat` → `/data` | Ephemeral (compose-managed) |
| Network | External overlay `outsideworx` | External `services_default` |
| Domain | `tundedivat.com` (no `www.` redirect) | `tunde-divat.localhost` |
| Health check | `/metrics` on port 80, interval 1m | `/metrics` on port 80, interval 5s |
| Metrics endpoint | `/metrics` returns `up 1` | `/metrics` returns `up 1` |
| Port | Exposes port 80 directly | Exposes port 80 directly |

## CI/CD (GitHub Actions)

See the `github-actions` prompt for the full pipeline description (build triggers, dispatch payload, deploy workflow).

Two distinct deployment paths exist:

- **Auto-deploy (site content)** — A push to a site repo dispatches the `build` job, which builds the one image, pushes it to GHCR, and then force-updates the running Swarm service (`docker service update --force sites_<name>`). Site content changes reach production with no manual step. Combined with the `start-first` / `rollback` update strategy above, this is a zero-downtime rolling update.
- **Manual deploy (stack changes)** — `deploy.sh` (run via the manual `deploy.yaml` workflow) is only needed for stack-level changes: `compose.yaml`, `.env`, or adding/removing services. The matrix `build-sites` job (push to `sites/main`) rebuilds all images to GHCR but does **not** update running services — a manual deploy applies them.

## Sites List

| Name | Domain | Type | Has API Token | Cache Volume |
|------|--------|------|---------------|--------------|
| come-in-and-find-out | come-in-and-find-out.ch | static | Yes | `services_cache` → `/htdocs/cache` (ro) |
| duckumbrella | duckumbrella.net | static | No | — |
| gaiapeeps | gaiapeeps.com | static | Yes | — |
| igli | igli.info | static | No | — |
| outsideworx | outsideworx.net | static | Yes | — |
| soupart | soupart.net | static | Yes | `services_cache` → `/htdocs/cache` (ro) |
| soupkitchen | soupkitchen.info | static | No | — |
| tunde-divat | tundedivat.com | npm | No (self-contained) | Named volume `tunde-divat` → `/data` (SQLite + uploads) |

The cache volume is a Docker named volume (`services_cache`, external — created by the services stack's `utils` container as `cache`). It is no longer a host bind mount; nothing under `/home/outsideworx/utils` is involved anymore. Static sites that serve cached images (`come-in-and-find-out`, `soupart`) mount the whole volume read-only at `/htdocs/cache` (not a per-client subdirectory). In test, both sites share the same named volume mounted at `/htdocs/cache`. The `tunde-divat` volume is a separate Docker named volume holding the app's own SQLite database and uploaded images — not a cache of the services DB.

## Fallback / Mirror URLs (`/clients/<name>`)

Every **static** (non-npm) site is also reachable as a mirror under the `outsideworx` site at `outsideworx.net/clients/<name>` (e.g. `outsideworx.net/clients/soupart`). This is a safety net: if an official domain is discontinued or its DNS/TLS lapses, the content is still served from the `outsideworx` container.

| Site | Official domain | Mirror URL |
|------|-----------------|------------|
| come-in-and-find-out | come-in-and-find-out.ch | `outsideworx.net/clients/come-in-and-find-out` |
| duckumbrella | duckumbrella.net | `outsideworx.net/clients/duckumbrella` |
| gaiapeeps | gaiapeeps.com | `outsideworx.net/clients/gaiapeeps` |
| igli | igli.info | `outsideworx.net/clients/igli` |
| soupart | soupart.net | `outsideworx.net/clients/soupart` |
| soupkitchen | soupkitchen.info | `outsideworx.net/clients/soupkitchen` |

No npm site is mirrored — an npm site is a self-contained dynamic app (its own server + database) with its own container, not static content that can be copied into the `outsideworx` image.

### How the Mirror Works (git submodules)

Each static site repo is added as a **git submodule** of the `outsideworx` repo under `clients/<name>/` (the same mechanism the WIP `thegreen` site uses — see `sites-wip.md`):

```
outsideworx repo
├── .gitmodules              # Declares one submodule per static site (bare name, no clients/ prefix)
└── clients/
    ├── come-in-and-find-out/  # Submodule → github.com/outsideworx/come-in-and-find-out
    ├── duckumbrella/
    ├── gaiapeeps/
    ├── igli/
    ├── soupart/
    ├── soupkitchen/
    └── thegreen/              # WIP site (secret-gated)
```

At Docker build time the `outsideworx` image's `fetcher` stage clones the `outsideworx` repo and runs `git submodule update --init --depth 1`, so every submodule's content is checked out into `/usr/local/apache2/htdocs/clients/<name>/`. Apache's `MultiViews` + `DirectoryIndex` on `htdocs/clients/` then serve each site at `outsideworx.net/clients/<name>`. Submodule URLs in `.gitmodules` are HTTPS (`https://github.com/outsideworx/<name>.git`) so the anonymous build clone can initialize them without SSH credentials.

Because the mirror is a submodule pointer, its content only refreshes when the `outsideworx` repo bumps the pointer (`git submodule update --remote clients/<name>`, commit, push) and rebuilds — it is not automatically in lockstep with the live site's own deploy. Unlike `thegreen`, these mirrors are **not** secret-gated (`CLIENT_SECRET_PATH` covers only `/clients/thegreen/`), so they are publicly reachable.

## File Layout

```
sites/
├── .env                    # Token variables
├── .github/workflows/
│   ├── build.yaml          # Matrix + dispatch builds
│   └── deploy.yaml         # SSH deploy (manual)
├── blacklist.conf          # Prod IP blacklist
├── blacklist-test.conf     # Test placeholder
├── compose.yaml            # Prod stack (pulls from GHCR)
├── compose-test.yaml       # Local dev (builds from Dockerfile.test)
├── Dockerfile              # Prod multi-site static image (Apache)
├── Dockerfile.test         # Test static variant (different proxy, relaxed limits)
├── Dockerfile.npm          # Prod npm-app image (tunde-divat: Node/Express + Vite SPA)
├── Dockerfile.npm.test     # Test npm-app variant
└── secret.lua.tpl          # Client secret Lua template
```
