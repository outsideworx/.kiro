# Sites Deployment

## Overview

The sites stack serves multiple websites, each built into its own container and routed by domain. Most sites are **static** (Apache httpd serving HTML/CSS/JS) and proxy API calls back to the services stack. One site (`tunde-divat`) is a **dynamic npm app** (Node/Express + Vite SPA with its own SQLite database) built from a separate Dockerfile.

## Architecture

```
GitHub (site repo) → repository_dispatch → sites repo CI → Docker build → GHCR
                                                                            ↓
Server: docker stack deploy → pulls images → Traefik routes by domain
```

Each site is a separate GitHub repository. Static sites contain HTML/CSS/JS (often Hugo-generated); the npm app is a full npm-workspaces monorepo. The `sites` repo provides the shared Dockerfiles and deployment config.

## Build Types (Dockerfile Selection)

The `sites` repo ships two families of Dockerfile, selected per site by a build **type**:

| Type | Dockerfile (prod / test) | Base | Serves | Sites |
|------|--------------------------|------|--------|-------|
| (default, no type) | `Dockerfile` / `Dockerfile.test` | `httpd` (Apache) | Static content, proxies `/api/` | all static sites |
| `npm` | `Dockerfile.npm` / `Dockerfile.npm.test` | `node:22-alpine` | Its own Express API + built SPA | `tunde-divat` |

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

## Dockerfile — npm App (`tunde-divat`)

`Dockerfile.npm` (prod) and `Dockerfile.npm.test` (test) build the `tunde-divat` dynamic app. Both are currently identical. The `NAME` build arg still selects the repo to clone, so the template stays multi-site-capable even though only one npm site exists today.

### Build Stages

1. **fetcher** (`bitnami/git`) — Clones `https://github.com/outsideworx/${NAME}.git` (depth 1)
2. **builder** (`node:22-alpine`) — `npm ci`, then `npm run build` with `VITE_API_URL=""` (the SPA calls the API on its own origin under `/api/`)
3. **runtime** (`node:22-alpine`) — Copies the built workspace, sets `WEB_DIST_DIR=/app/apps/web/dist` and `NODE_ENV=production`, generates an entrypoint script

### Entrypoint

The generated `/app/entrypoint.sh` runs, from `/app/apps/api`:

```sh
npx prisma migrate deploy   # apply SQLite migrations
npm run seed                # seed admin user + invite code
exec node dist/server.js    # start Express on API_PORT
```

### Runtime Characteristics

- **Port**: exposes and listens on `4000` (not 80) — the Express app serves both the API (`/api/`) and the built SPA (`WEB_DIST_DIR`)
- **No Apache**: no proxy, no `TOKEN`/`X-Auth-Token` injection, no blacklist, no MPM/rate-limit config — all of that lives inside the Express app
- **No `/api/` proxy to services**: the app is self-contained; it does not call the Spring Boot backend
- **Persistence**: a SQLite DB file and uploads directory live on a named volume mounted at `/data` (`DATABASE_URL=file:/data/tunde-divat.db`, `UPLOAD_DIR=/data/uploads`)
- **Health check**: `/api/health` on port 4000 (static sites use `/metrics` on port 80)
- **No `/metrics`**: the app is not a Prometheus scrape target (see the `monitoring` prompt)

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
| `TUNDE_DIVAT_AI_PROVIDER` | tunde-divat | AI image provider (`openai` or `mock`) → `AI_PROVIDER` |
| `TUNDE_DIVAT_OPENAI_API_KEY` | tunde-divat | OpenAI API key → `OPENAI_API_KEY` (only used when provider is `openai`) |
| `TUNDE_DIVAT_SEED_ADMIN_PASSWORD` | tunde-divat | Seeded admin password → `SEED_ADMIN_PASSWORD` |
| `TUNDE_DIVAT_SEED_ADMIN_USERNAME` | tunde-divat | Seeded admin username → `SEED_ADMIN_USERNAME` |
| `TUNDE_DIVAT_SEED_INVITE_CODE` | tunde-divat | Registration invite code → `SEED_INVITE_CODE` |
| `TUNDE_DIVAT_SESSION_SECRET` | tunde-divat | JWT/session signing secret (≥32 chars) → `SESSION_SECRET` |

Only static sites that call the API need a `TOKEN`. Static sites without API calls (duckumbrella, igli, outsideworx, soupkitchen) have no `TOKEN` environment variable. The `tunde-divat` npm app uses none of the `TOKEN`/`CLIENT_SECRET` mechanism — it has its own `TUNDE_DIVAT_*` variables (mapped to the Express app's env in compose). `DATABASE_URL`, `UPLOAD_DIR`, and `CORS_ORIGIN` are set as literals in `compose.yaml`, not via `.env`.

The `outsideworx` service is deployed in `global` mode (one instance per Swarm node) rather than replicated mode. On the current single-node cluster this is functionally equivalent to one replica, but global mode ensures an instance is present on every node without specifying a replica count — useful if the cluster ever scales out.

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
| `AI_PROVIDER` | From `.env` (`TUNDE_DIVAT_AI_PROVIDER`) | `openai` (hardcoded) |
| `OPENAI_API_KEY` | From `.env` (`TUNDE_DIVAT_OPENAI_API_KEY`) | `""` (empty) |
| `CORS_ORIGIN` | `https://tundedivat.com` | `https://tunde-divat.localhost` |
| Seed vars | From `.env` (`TUNDE_DIVAT_SEED_*`) | Hardcoded test values |
| `SESSION_SECRET` | From `.env` (`TUNDE_DIVAT_SESSION_SECRET`) | Hardcoded 32+ char test string |
| Volume | Named volume `tunde-divat` → `/data` | Ephemeral (compose-managed) |
| Network | External overlay `outsideworx` | External `services_default` |
| Domain | `tundedivat.com` (no `www.` redirect) | `tunde-divat.localhost` |
| Health check | `/api/health` on `:4000`, interval 1m | `/api/health` on `:4000`, interval 5s |

## CI/CD (GitHub Actions)

See the `github-actions` prompt for the full pipeline description (build triggers, dispatch payload, deploy workflow).

## Sites List

| Name | Domain | Type | Has API Token | Cache Volume |
|------|--------|------|---------------|--------------|
| come-in-and-find-out | come-in-and-find-out.ch | static | Yes | `/home/outsideworx/utils/cache/ciafo` → `/htdocs/cache/ciafo` |
| duckumbrella | duckumbrella.net | static | No | — |
| gaiapeeps | gaiapeeps.com | static | Yes | — |
| igli | igli.info | static | No | — |
| outsideworx | outsideworx.net | static | No | — |
| soupart | soupart.net | static | Yes | `/home/outsideworx/utils/cache/soup` → `/htdocs/cache/soup` |
| soupkitchen | soupkitchen.info | static | No | — |
| tunde-divat | tundedivat.com | npm | No (self-contained) | Named volume `tunde-divat` → `/data` (SQLite + uploads) |

The cache volumes are host bind mounts from the path written by the `utils` container. Static sites that serve cached images (`come-in-and-find-out`, `soupart`) mount the relevant subdirectory read-only. In test, both sites share a single named volume (`services_cache`) mounted at `/htdocs/cache`. The `tunde-divat` volume is a Docker named volume holding the app's own SQLite database and uploaded images — not a cache of the services DB.

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
