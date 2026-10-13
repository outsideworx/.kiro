# Traefik Reverse Proxy

## Overview

Traefik v3 is the single entry point for all HTTP/HTTPS traffic. It handles TLS termination, routing, and service discovery via Docker labels.

- Image: `traefik`
- Ports: 80 (HTTP), 443 (HTTPS), 81 (Traefik dashboard/metrics)
- Network: `outsideworx` and `edge` overlays (Traefik joins both — see `networking.md`)

## Entrypoints

| Name | Port | Purpose |
|------|------|---------|
| `web` | 80 | HTTP — auto-redirects to `websecure` |
| `websecure` | 443 | HTTPS — all application traffic |
| `traefik` | 81 | Internal dashboard and Prometheus metrics |

## TLS / Certificates

- Prod: Let's Encrypt via HTTP challenge (`certresolver=letsencrypt`)
- ACME storage: `/letsencrypt/acme.json` (persistent volume on host)
- ACME email: `info@outsideworx.net`
- Test: Self-signed (Traefik default), labels use `tls=true` instead of `tls.certresolver`

## Service Discovery

### Prod (Swarm)

```
--providers.swarm=true
--providers.swarm.endpoint=unix:///var/run/docker.sock
--providers.swarm.exposedbydefault=false
```

Labels go in `deploy.labels` (Swarm reads deploy section).

### Test (Compose)

```
--providers.docker=true
--providers.docker.exposedbydefault=false
```

Labels go at service level (Compose reads top-level labels).

## Label Convention

Every routed service needs these labels:

```yaml
# Prod (in deploy.labels)
- traefik.enable=true
- traefik.http.routers.<name>.entrypoints=websecure
- traefik.http.routers.<name>.rule=Host(`<domain>`)
- traefik.http.routers.<name>.tls.certresolver=letsencrypt
- traefik.http.services.<name>.loadbalancer.server.port=80

# Test (in labels)
- traefik.enable=true
- traefik.http.routers.<name>.entrypoints=websecure
- traefik.http.routers.<name>.rule=Host(`<name>.localhost`)
- traefik.http.routers.<name>.tls=true
- traefik.http.services.<name>.loadbalancer.server.port=80
```

### Optional Labels

- Health check: `traefik.http.services.<name>.loadbalancer.healthcheck.path=/metrics`
- Health interval: `traefik.http.services.<name>.loadbalancer.healthcheck.interval=1m` (prod) / `5s` (test)
- Middleware: `traefik.http.routers.<name>.middlewares=<middleware-name>`

### Middlewares

Currently defined:
- `compress` — gzip/brotli response compression for text content types (used by all sites, incl. tunde-divat)
- `www-redirect` — Strips `www.` prefix via regex redirect (used by all static sites)

```yaml
- traefik.http.middlewares.compress.compress.includedcontenttypes=text/css,text/html,text/javascript,text/plain,application/javascript,application/json,image/svg+xml
- traefik.http.middlewares.www-redirect.redirectregex.permanent=true
- traefik.http.middlewares.www-redirect.redirectregex.regex=^https://www\.(.+)
- traefik.http.middlewares.www-redirect.redirectregex.replacement=https://$${1}
```

Both are defined once, on the `outsideworx` site service's labels in `sites/compose.yaml` (`compose-test.yaml` defines only `compress`), and referenced as `<name>@swarm` / `<name>` from every router. `includedContentTypes` keeps Traefik from recompressing images/video; already-encoded responses are passed through. Compression is applied in Traefik rather than Apache (`mod_deflate` is not enabled) so it also covers the Express app.

## Routing Map

### Services Stack

| Service | Domain | Port |
|---------|--------|------|
| authelia | `oauth.outsideworx.net` | 80 |
| grafana | `grafana.outsideworx.net` | 80 |
| ntfy | `ntfy.outsideworx.net` | 80 |
| services | `services.outsideworx.net` | 80 |
| wg-easy | `vpn.outsideworx.net` | 51821 (admin UI only — see the `wg-easy` prompt) |

### Sites Stack

| Site | Domain(s) | Middleware |
|------|-----------|------------|
| come-in-and-find-out | `come-in-and-find-out.ch`, `www.come-in-and-find-out.ch` | compress, www-redirect |
| duckumbrella | `duckumbrella.net`, `www.duckumbrella.net` | compress, www-redirect |
| gaiapeeps | `gaiapeeps.com`, `www.gaiapeeps.com` | compress, www-redirect |
| igli | `igli.info`, `www.igli.info` | compress, www-redirect |
| outsideworx | `outsideworx.net`, `www.outsideworx.net` | compress, www-redirect |
| soupart | `soupart.net`, `www.soupart.net` | compress, www-redirect |
| soupkitchen | `soupkitchen.info`, `www.soupkitchen.info` | compress, www-redirect |
| tunde-divat | `tundedivat.com` | compress |

All static sites listen on port `80` and use the `/metrics` health check. npm apps are also served on port `80` with a `/metrics` health check. The main differences from static sites are:
- Single-host rule (no `www.` redirect)
- No `www.` alternate domain

Every static (non-npm) site is additionally reachable as a fallback mirror at `outsideworx.net/clients/<name>` — served statically from the `outsideworx` container, where each static site repo is checked out as a git submodule under `clients/<name>/` and served by Apache from `htdocs/clients/`. Traefik is not involved beyond matching the existing `outsideworx` host; the mirror content is plain static files, not a proxied backend. If an official domain is discontinued, the mirror still works. No npm site is mirrored (npm sites are self-contained dynamic apps, not static content that can be copied into the `outsideworx` image).

## Prod vs Test

| Aspect | Prod (Swarm) | Test (Compose) |
|--------|--------------|----------------|
| Provider | `--providers.swarm` | `--providers.docker` |
| TLS | Let's Encrypt (`certresolver=letsencrypt`) | Self-signed (`tls=true`) |
| Domains | Real domains (`.net`, `.ch`, `.com`, `.info`) | `*.localhost` |
| Labels location | `deploy.labels` | Service-level `labels` |
| Port binding | `mode: host` (bypasses ingress) | Standard port mapping |
| Access log | Disabled | Enabled (`--accesslog=true`) |
| Health check interval | 1m | 5s |
| Metrics | `--metrics.prometheus=true` | `--metrics.prometheus=true` |

## Adding a New Routed Service

1. Add Traefik labels to the service in `compose.yaml` (prod) and `compose-test.yaml` (test)
2. Ensure the service is on `outsideworx` or `edge` (whichever it needs — Traefik reaches both; see `networking.md`)
3. Set `traefik.enable=true` and configure router + service labels
4. For TLS: use `certresolver=letsencrypt` (prod) or `tls=true` (test)
5. Attach the `compress` middleware (`middlewares=compress`, or `compress,www-redirect`)
6. If the domain needs `www.` redirect, attach the `www-redirect` middleware
