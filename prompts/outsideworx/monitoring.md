# Monitoring Stack

## Components

| Service | Image | Port | Metrics Port |
|---------|-------|------|--------------|
| Prometheus | prom/prometheus:v3.9.1 | — | internal |
| Grafana | grafana/grafana:12.3.0 | 80 | — |
| Loki | grafana/loki:3.6.4 | 80 | — |
| Promtail | grafana/promtail:3.6.4 | 80 | 80 |
| ntfy | binwiederhier/ntfy:v2.16.0 | 80 | 81 |

## Prometheus

- Config: `prometheus.yaml` (prod), `prometheus-test.yaml` (test)
- Scrape interval: 1m (prod), 5s (test)
- Retention: 365 days (`--storage.tsdb.retention.time=365d`)
- Data volume: `/home/outsideworx/prometheus`
- Scrape targets: authelia:81, loki, ntfy:81, postgres-exporter, promtail, services:81 (path `/actuator/prometheus`), all site containers, traefik:81
- In test mode, services target is `host.docker.internal:81` (app runs on host)

## Grafana

- Config: `grafana.ini` (prod), `grafana-test.ini` (test)
- Exposed at: `grafana.outsideworx.net`
- Auth: Authelia OIDC (auto-login, no login form)
- Role mapping: `groups && contains(groups, 'admin') && 'GrafanaAdmin' || 'Viewer'`
- Data volume: `/home/outsideworx/grafana`
- Runs as root (volume permissions)

## Loki

- Config: `loki.yaml`
- Schema: v13, TSDB store, filesystem object store
- Retention: 365 days
- Auth disabled (single-tenant)
- Data volume: `/home/outsideworx/loki`
- Runs as root (volume permissions)
- In test mode, port 3100 is exposed to host

## Promtail

- Config: `promtail.yaml` (prod), `promtail-test.yaml` (test)
- Deploy mode: global (runs on every swarm node)
- Discovers containers via Docker socket
- Pushes to `http://loki/loki/api/v1/push` (prod) or `http://loki:3100/...` (test)
- Positions file: `/promtail/positions.yaml`
- Pipeline stages:
  1. Extract log level via regex
  2. Normalize `ERR`/`err` → `ERROR`
  3. Strip timestamps, level keywords, and redundant whitespace from log lines
- Relabeling (prod — Swarm):
  - `job` = stack name (prefix before `_` in service name)
  - `app` = service name (suffix after `_`)
- Relabeling (test — Compose):
  - `job` = compose project name
  - `app` = container name (stripped leading `/`)

## ntfy

- Config: `ntfy.yaml` (prod), `ntfy-test.yaml` (test)
- Exposed at: `ntfy.outsideworx.net`
- Auth: required login, deny-all default, user DB at `/etc/ntfy/data/user.db`
- Metrics on port 81
- Cache: 365 days, file-backed at `/etc/ntfy/data/cache.db`
- Web push enabled (prod), key pair via environment variables
- Data volume: `/home/outsideworx/ntfy` (shared with utils container for DB access)

## Prod vs Test Differences

| Aspect | Prod (Swarm) | Test (Compose) |
|--------|--------------|----------------|
| Promtail refresh | 1m | 5s |
| Prometheus scrape | 1m | 5s |
| Loki URL | `http://loki/loki/api/v1/push` | `http://loki:3100/loki/api/v1/push` |
| Promtail relabeling | Swarm service labels | Compose container names |
| Grafana root URL | `grafana.outsideworx.net` | `grafana.localhost` |
| ntfy auth | env-based keys | hardcoded test token |
| Services target | `services:81` | `host.docker.internal:81` |

## File Layout

```
services/
├── grafana.ini              # Prod Grafana config
├── grafana-test.ini         # Test Grafana config
├── loki.yaml                # Loki config (shared)
├── ntfy.yaml                # Prod ntfy config
├── ntfy-test.yaml           # Test ntfy config
├── prometheus.yaml          # Prod Prometheus config
├── prometheus-test.yaml     # Test Prometheus config
├── promtail.yaml            # Prod Promtail config
└── promtail-test.yaml       # Test Promtail config
```

## Adding a New Scrape Target

1. Add entry to `prometheus.yaml` under `scrape_configs`
2. Add corresponding entry to `prometheus-test.yaml` (adjust host if needed)
3. Ensure the service exposes a `/metrics` endpoint (or specify `metrics_path`)
4. The service must be on the `outsideworx` overlay network
