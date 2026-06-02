# Traefik Reverse Proxy

## Overview

Traefik v3 is the single entry point for all HTTP/HTTPS traffic. It handles TLS termination, routing, and service discovery via Docker labels.

- Image: `traefik:v3.6.14`
- Ports: 80 (HTTP), 443 (HTTPS), 81 (Traefik dashboard/metrics)
- Network: `outsideworx` overlay (shared by all services)

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
- `www-redirect` — Strips `www.` prefix via regex redirect (used by all sites)

```yaml
- traefik.http.middlewares.www-redirect.redirectregex.permanent=true
- traefik.http.middlewares.www-redirect.redirectregex.regex=^https://www\.(.+)
- traefik.http.middlewares.www-redirect.redirectregex.replacement=https://$${1}
```

## Routing Map

### Services Stack

| Service | Domain | Port |
|---------|--------|------|
| authelia | `oauth.outsideworx.net` | 80 |
| grafana | `grafana.outsideworx.net` | 80 |
| ntfy | `ntfy.outsideworx.net` | 80 |
| services | `services.outsideworx.net` | 80 |

### Sites Stack

| Site | Domain(s) | Middleware |
|------|-----------|------------|
| come-in-and-find-out | `come-in-and-find-out.ch`, `www.come-in-and-find-out.ch` | www-redirect |
| duckumbrella | `duckumbrella.net`, `www.duckumbrella.net` | www-redirect |
| gaiapeeps | `gaiapeeps.com`, `www.gaiapeeps.com` | www-redirect |
| igli | `igli.info`, `www.igli.info` | www-redirect |
| outsideworx | `outsideworx.net`, `www.outsideworx.net` | www-redirect |
| soupart | `soupart.net`, `www.soupart.net` | www-redirect |
| soupkitchen | `soupkitchen.info`, `www.soupkitchen.info` | www-redirect |

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
2. Ensure the service is on the `outsideworx` overlay network
3. Set `traefik.enable=true` and configure router + service labels
4. For TLS: use `certresolver=letsencrypt` (prod) or `tls=true` (test)
5. If the domain needs `www.` redirect, attach the `www-redirect` middleware
