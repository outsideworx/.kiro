# Networking

## Overlay Network

A single Docker overlay network (`outsideworx`) connects all containers across both stacks. It is external and attachable — created once during initial Swarm setup (`deploy.sh --network`).

```yaml
networks:
  outsideworx:
    external: true
```

Both `services` and `sites` stacks attach to this network. There are no per-service networks.

Because the network is shared, any container can reach any other container regardless of which stack it belongs to. This enables:
- Traefik (in `services` stack) routing traffic to site containers (in `sites` stack)
- Site containers (in `sites` stack) proxying API requests to the services app (in `services` stack)
- Prometheus (in `services` stack) scraping metrics from all site containers (in `sites` stack)

## Swarm VIP DNS (Prod)

Docker Swarm assigns each service a Virtual IP (VIP) and registers it in the overlay network's internal DNS under the format:

```
<stack>_<service>
```

Examples: `services_postgres`, `services_authelia`, `sites_soupart`

This name resolves globally across all nodes in the Swarm — any container on the `outsideworx` network can reach any other service by its VIP DNS name regardless of which node the target runs on.

## Network Aliases (Java/Spring Boot RFC Compliance)

Java and Spring Boot strictly enforce RFC 3986, which forbids underscores in hostnames. This affects the Spring Boot app in two directions — both when it calls other services and when it receives requests. All aliases exist solely because of this strict compliance; no other service in the stack has this problem.

### Outbound Aliases

The Spring Boot app needs to *call out* to `postgres` (JDBC) and `authelia` (OIDC HTTP). Java's `URI` class rejects the underscore VIP names, so these services have dash-format aliases that the app uses in `application.yaml`:

```yaml
# compose.yaml
authelia:
  networks:
    outsideworx:
      aliases:
        - services-authelia

postgres:
  networks:
    outsideworx:
      aliases:
        - services-postgres
```

### Inbound Alias

The Spring Boot app *receives* scrape requests from Prometheus. When Prometheus targets `services_services:81`, it sends `Host: services_services` in the request header. Spring Boot's host header validation rejects the underscore. The alias on the services container itself gives Prometheus a valid hostname to use:

```yaml
# compose.yaml
services:
  networks:
    outsideworx:
      aliases:
        - services-services
```

Prometheus scrape config uses `services-services:81` as the target.

### Why Not Aliases Everywhere?

Aliases are task-scoped in Swarm — they only resolve on the node where the container's task runs. VIP DNS names resolve globally. Using aliases for Promtail (deployed in `global` mode) would break resolution on worker nodes. The underscore format works for everything except Java/Spring Boot's strict RFC compliance.

## Hostname Conventions

| Context | Format | Example |
|---------|--------|---------|
| Prod config referencing a service | `<stack>_<service>` | `services_postgres`, `services_loki` |
| Java code (application.yaml) | `services-<service>` (alias) | `services-postgres`, `services-authelia` |
| Prometheus scraping Spring Boot | `services-services` (alias) | `services-services:81` |
| Test config (Docker Compose) | Short container name | `postgres`, `authelia`, `loki` |
| Test config (host services) | `localhost` or `host.docker.internal` | `localhost:5432`, `host.docker.internal:81` |

## Test Network (Docker Compose)

In test mode, Docker Compose creates a default bridge network (`services_default`). Containers resolve each other by their `container_name`. The Spring Boot app runs on the host (not in a container), so:

- It reaches PostgreSQL at `localhost:5432` (port exposed to host)
- It reaches Authelia at `https://oauth.localhost` (via Traefik on host port 443)
- Other containers reach the host app at `host.docker.internal:8080`

The sites test stack (`compose-test.yaml`) joins the `services_default` network as external:

```yaml
networks:
  services_default:
    external: true
```

## Service Communication Graph

```
+----------------------------------------------------------------------------------------------------------+
|  outsideworx overlay network                                                                             |
|                                                                                                          |
|  Internet                                                                                                |
|      |                                                                                                   |
|      v                                                                                                   |
|  +----------------------------------------------------------------------------------------------------+  |
|  |                                    traefik :80, :443, :81                                          |  |
|  |            Routes by Host header to ALL labeled services (TLS termination)                         |  |
|  |  Backends: authelia, grafana, ntfy, services, all sites (come-in-and-find-out .. soupkitchen)      |  |
|  +----------------------------------------------------------------------------------------------------+  |
|                                                                                                          |
|                                                                                                          |
|  +-----------------+            +-----------------+            +-----------------+                       |
|  |    authelia     |<---OIDC----|    services     |---JDBC---->|    postgres     |                       |
|  |    :80, :81     |            |    :80, :81     |            |     :5432       |                       |
|  +--------+--------+            +--------+--------+            +--------+--------+                       |
|           |                              |                              |                                |
|           | OIDC                         |                              |                                |
|           v                              |                              |                                |
|  +-----------------+                     |                              |                                |
|  |     grafana     |                     |                              |                                |
|  |      :80        |                     |                              |                                |
|  +-----------------+                     |                              |                                |
|                                          |                              |                                |
|  +-----------------+                     |                              |                                |
|  |      ntfy       |                     |                              |                                |
|  |    :80, :81     |                     |                              |                                |
|  +-----------------+                     |                              |                                |
|                                          |                              |                                |
|  +------------------------------+        |                              |                                |
|  |     sites (Apache :80 each)  |--------+                              |                                |
|  |     ProxyPass /api/ ------------>  services                          |                                |
|  +------------------------------+                                       |                                |
|                                                                         |                                |
|  +------------------------------+                                       |                                |
|  |     postgres-exporter :80    |---------------------------------------+                                |
|  +------------------------------+                                                                        |
|                                                                                                          |
|  +------------------------------+                                                                        |
|  |     utils (cache.py)         |------PostgreSQL----> postgres :5432                                    |
|  +------------------------------+                                                                        |
|                                                                                                          |
|  +------------------------------+                                                                        |
|  |     prometheus               |  Scrapes ALL services that expose metrics                              |
|  +------------------------------+                                                                        |
|                                                                                                          |
|  +------------------------------+         +-----------------+                                            |
|  |     promtail (global)        |-------->|      loki       |  Collects logs from ALL containers         |
|  |                              |  push   |      :80        |                                            |
|  +------------------------------+         +-----------------+                                            |
+----------------------------------------------------------------------------------------------------------+
```

### Reading the Graph

- **Traefik** is the single ingress point. It terminates TLS and routes to all labeled backend services based on the `Host` header: 4 in the services stack (authelia, grafana, ntfy, services) and all sites in the sites stack (each on its own domain).
- **Sites → services** is an internal connection (Apache `ProxyPass`), not routed through Traefik. Only the 3 sites with API tokens (come-in-and-find-out, gaiapeeps, soupart) make these calls.
- **Grafana → authelia** is also internal (OIDC token exchange), not through Traefik.
- **Prometheus** scrapes every service that exposes metrics — authelia:81, loki, ntfy:81, postgres-exporter, promtail, services:81, traefik:81, and all sites.
- **Promtail** runs in global mode (one instance per Swarm node) and pushes logs from all containers to Loki.

## Detailed Communication Paths

### Prod (Swarm VIP DNS)

| Source | Target | Protocol | Purpose |
|--------|--------|----------|---------|
| services | services-postgres (alias) | JDBC :5432 | Database |
| services | services-authelia (alias) | HTTP :80 | OIDC token, userinfo, JWKS |
| grafana | services_authelia | HTTP :80 | OIDC token, userinfo |
| prometheus | services_authelia | HTTP :81 | Metrics scrape |
| prometheus | services_loki | HTTP :80 | Metrics scrape |
| prometheus | services_ntfy | HTTP :81 | Metrics scrape |
| prometheus | services_postgres-exporter | HTTP :80 | Metrics scrape |
| prometheus | services_promtail | HTTP :80 | Metrics scrape |
| prometheus | services-services (alias) | HTTP :81 | Metrics scrape (actuator) |
| prometheus | services_traefik | HTTP :81 | Metrics scrape |
| prometheus | sites_* | HTTP :80 | Metrics scrape (all sites) |
| promtail | services_loki | HTTP :80 | Log push (`/loki/api/v1/push`) |
| postgres-exporter | services_postgres | PostgreSQL :5432 | DB metrics |
| utils (cache.py) | services_postgres | PostgreSQL :5432 | Image sync |
| sites (Apache) | services_services | HTTP :80 | API proxy (`/api/`) |
| traefik | all labeled services | HTTP :80 | Reverse proxy routing |

### Test (Docker Compose Short Names)

| Source | Target | Protocol | Purpose |
|--------|--------|----------|---------|
| services (host) | localhost | JDBC :5432 | Database |
| services (host) | oauth.localhost (via Traefik) | HTTPS :443 | OIDC (browser redirects) |
| services (host) | oauth.localhost (via Traefik) | HTTPS :443 | OIDC token/userinfo/JWKS |
| grafana | authelia | HTTP :80 | OIDC token, userinfo |
| prometheus | authelia | HTTP :81 | Metrics scrape |
| prometheus | loki | HTTP :3100 | Metrics scrape |
| prometheus | ntfy | HTTP :81 | Metrics scrape |
| prometheus | postgres-exporter | HTTP :80 | Metrics scrape |
| prometheus | promtail | HTTP :80 | Metrics scrape |
| prometheus | host.docker.internal | HTTP :81 | Metrics scrape (services on host) |
| prometheus | sites (short names) | HTTP :80 | Metrics scrape |
| promtail | loki | HTTP :3100 | Log push |
| postgres-exporter | postgres | PostgreSQL :5432 | DB metrics |
| utils (cache.py) | postgres | PostgreSQL :5432 | Image sync |
| sites (Apache) | host.docker.internal | HTTP :8080 | API proxy (`/api/`) |

## Port Conventions

| Port | Used for |
|------|----------|
| 80 | Default HTTP (most services) |
| 81 | Metrics/actuator (authelia, ntfy, services, traefik) |
| 443 | HTTPS (Traefik only) |
| 3100 | Loki (test only — prod uses port 80) |
| 5432 | PostgreSQL |
| 8080 | Spring Boot app in test mode (on host) |

## Key Rules

1. **Never use short names in prod configs** — always `<stack>_<service>` format
2. **Never use `services_` prefix in test configs** — always short container names
3. **Only add network aliases when Java/Spring Boot is involved** — outbound (`authelia`, `postgres`) or inbound (`services` itself)
4. **Aliases are task-scoped** — they don't resolve across nodes; VIP DNS does
5. **All services share one flat network** — no network segmentation, no service-specific networks
6. **Loki uses port 80 in prod, 3100 in test** — test exposes 3100 to host for debugging
