# Sites Deployment

## Overview

The sites stack serves multiple static websites, each built into its own Apache httpd container from a shared Dockerfile. All sites proxy API calls back to the services stack.

## Architecture

```
GitHub (site repo) → repository_dispatch → sites repo CI → Docker build → GHCR
                                                                            ↓
Server: docker stack deploy → pulls images → Traefik routes by domain
```

Each site is a separate GitHub repository containing static HTML/CSS/JS (often Hugo-generated). The `sites` repo provides the shared Dockerfile and deployment config.

## Dockerfile (Multi-Site, Single Template)

The same `Dockerfile` builds all sites. The `NAME` build arg determines which site repo to clone.

### Build Stages

1. **fetcher** (`bitnami/git`) — Clones `https://github.com/outsideworx/${NAME}.git` with depth 1, initializes submodules
2. **runtime** (`httpd:2.4`) — Copies site content, configures Apache

### Apache Configuration

Modules enabled: `headers`, `negotiation`, `proxy`, `proxy_http`, `ratelimit`, `remoteip`, `reqtimeout`, `unique_id`

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
RedirectMatch 403 ^(?!/cache/).*\.(bak|conf|config|env|ini|json|key|log|properties|php|pub|py|sh|ts|yaml|yml|zip)/?$
RedirectMatch 403 ^(?!/(cache/|metrics\.txt|robots\.txt)).*\.txt/?$
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

Only sites that call the API need a token. Sites without API calls (duckumbrella, igli, outsideworx, soupkitchen) have no `TOKEN` environment variable.

## Client Secret Access Control

A lightweight protection mechanism for work-in-progress sites that need to share demo URLs with clients without exposing unfinished content to the public. Avoids the overhead of full OAuth2 — just "enter secret to view."

### How It Works

```
Request to ${CLIENT_SECRET_PATH}page
        │
        ▼
LuaHookAccessChecker (secret.lua)
        │
        ├── Has cookie access_token=granted? → Pass through (DECLINED)
        │
        └── No cookie → 302 redirect to ${CLIENT_SECRET_PATH}secret
                                │
                                ▼
                        Renders HTML form (dark-themed, password input)
                                │
                                ▼ POST
                        Compare submitted password to CLIENT_SECRET
                                │
                        ├── Match → Set cookie, 302 to protected path
                        └── Mismatch → Re-render form with error
```

The template (`secret.lua.tpl`) has two placeholders: `{{CLIENT_SECRET}}` and `{{CLIENT_SECRET_PATH}}`.

- **Cookie attributes**: `HttpOnly` (no JS access), `SameSite=Strict` (no CSRF), `Path={CLIENT_SECRET_PATH}` (scoped to protected path)
- **No `Secure` flag**: works on HTTP in test mode
- **Session cookie**: no `Max-Age` or `Expires` — cleared when browser closes
- **No server-side state**: the cookie itself is the proof of access

At container startup, the entrypoint script (`docker-entrypoint.sh`) checks for `CLIENT_SECRET`:

1. If set: runs `sed` on `secret.lua.tpl` to inject `CLIENT_SECRET` and `CLIENT_SECRET_PATH`, producing `/usr/local/apache2/conf/secret.lua`
2. Generates `conf/extra/httpd-auth.conf` with:
   ```apache
   <Location "${CLIENT_SECRET_PATH}">
       LuaHookAccessChecker /usr/local/apache2/conf/secret.lua check_access
   </Location>
   ```
3. If not set: writes `# No auth` to `httpd-auth.conf` (no-op include)

Requires `mod_lua` (enabled in both Dockerfiles via `sed`).

### Configuration

Compose environment variables on the site service:

```yaml
environment:
  CLIENT_SECRET: $APP_CLIENTS_<CLIENT>_SECRET
  CLIENT_SECRET_PATH: "/path/to/protect/"
```

The path **must** end with `/`. All requests under that path require the secret.

### Adding Client Secret to Another Site

1. Add `CLIENT_SECRET` and `CLIENT_SECRET_PATH` to the site's environment in `compose.yaml`
2. Add the same in `compose-test.yaml` (hardcoded value for local dev)
3. Add `APP_CLIENTS_<CLIENT>_SECRET=<value>` to both `sites/.env` and `services/.env`
4. No Dockerfile changes needed — the entrypoint handles it automatically

## Prod vs Test

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
| Client secret | From `.env` via `$APP_CLIENTS_<CLIENT>_SECRET` | Hardcoded in `compose-test.yaml` |

## CI/CD (GitHub Actions)

### Build Triggers

1. **Push to main** — Matrix build: builds all sites in parallel, pushes to GHCR
2. **repository_dispatch** — Triggered by individual site repos when they update; builds only the changed site

### Dispatch Payload

Site repos send:
```json
{ "event_type": "build-<name>", "client_payload": { "name": "<name>" } }
```

### Deploy

Manual `workflow_dispatch` — SSH to host, git pull, run `deploy.sh`.

## Sites List

| Name | Domain | Has API Token |
|------|--------|---------------|
| come-in-and-find-out | come-in-and-find-out.ch | Yes |
| duckumbrella | duckumbrella.net | No |
| gaiapeeps | gaiapeeps.com | Yes |
| igli | igli.info | No |
| outsideworx | outsideworx.net | No |
| soupart | soupart.net | Yes |
| soupkitchen | soupkitchen.info | No |

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
├── Dockerfile              # Prod multi-site image
├── Dockerfile.test         # Test variant (different proxy, relaxed limits)
└── secret.lua.tpl          # Client secret Lua template
```
