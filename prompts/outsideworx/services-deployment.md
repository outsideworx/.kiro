# Services Deployment

## Overview

The services stack runs on Docker Swarm as a single-node cluster. It contains the Spring Boot app, PostgreSQL, monitoring, auth, and utility containers — all on a shared overlay network.

## Infrastructure

- Single manager node (all constraints: `node.role == manager`)
- Overlay network: `outsideworx` (external, attachable)
- Docker secret: `rsa_private_key` (RSA 4096-bit, used by Authelia for OIDC signing)
- Stack name: `services`

## deploy.sh

The deployment script handles both initial setup and updates.

### Flags

| Flag | Action |
|------|--------|
| `--install` | Installs `docker-compose-v2` via apt |
| `--network` | Initializes Swarm (`docker swarm init --advertise-addr <IP>`) and creates the overlay network |
| `--secrets` | Generates RSA 4096 key and stores as Docker secret |
| (no flag) | Deploys/updates the stack |

### Deployment Flow (no flag)

1. Creates `/home/outsideworx/services/` (if missing)
2. Creates `/home/outsideworx/utils` (bind mount for cache output)
3. Copies `utils/` directory and config files to deploy directory
4. Sources `.env` file
5. Computes SHA256 hashes of config files (for Swarm config naming)
6. `docker compose pull`
7. `docker stack deploy -c compose.yaml services --detach=false --resolve-image=always`
8. Force-updates all services (rolling restart)

### Prerequisites

- Docker Swarm initialized (`--network` flag, requires IP address)
- Overlay network created (`--network` flag)
- RSA secret created (`--secrets` flag)
- `.env` file present in repo root with all required variables

## Compose Structure (compose.yaml)

### Services

| Service | Image | Exposed | Notes |
|---------|-------|---------|-------|
| traefik | traefik | 80, 443 (host mode) | See traefik.md |
| authelia | authelia/authelia | via Traefik | See auth.md |
| services | ghcr.io/outsideworx/services:latest | via Traefik | Spring Boot app |
| postgres | postgres | internal only | Named volume `postgres` |
| grafana | grafana/grafana | via Traefik | See monitoring.md |
| loki | grafana/loki | internal only | See monitoring.md |
| prometheus | prom/prometheus | internal only | See monitoring.md |
| promtail | grafana/promtail | internal only | Global deploy mode |
| ntfy | binwiederhier/ntfy | via Traefik | See monitoring.md |
| postgres-exporter | quay.io/prometheuscommunity/postgres-exporter | internal only | |
| utils | python (slim) | internal only | See python-utils.md |

### Network

All services join the `outsideworx` external overlay network. No service-specific networks.

### Secrets

```yaml
secrets:
  rsa_private_key:
    external: true
```

Used only by the `authelia` service.

### Configs

Swarm configs inject configuration files into containers. Each config is named with a SHA256 hash suffix (e.g., `authelia-a1b2c3d4`) computed by `deploy.sh`. This forces Swarm to detect changes and recreate the config on redeploy — Swarm treats configs as immutable, so a new name means a new config.

```yaml
configs:
  authelia:
    file: ./authelia.yaml
    name: authelia-${HASH_AUTHELIA}
```

Config files: `authelia.yaml`, `authelia-users.yaml`, `grafana.ini`, `logo.png`, `loki.yaml`, `ntfy.yaml`, `prometheus.yaml`, `promtail.yaml`.

#### Adding a New Config

1. Add the config entry to `compose.yaml` under `configs:` with `name: <name>-${HASH_<NAME>}`
2. Add `export HASH_<NAME>=$(sha256sum <file> | cut -c1-8)` to `deploy.sh`
3. Add the file to the `cp` command in `deploy.sh`
4. Mount it in the service via `configs:` with `source` and `target`

### Volumes

Named Docker volumes for persistent data. Survive stack removal and redeployment.

```yaml
volumes:
  grafana:
  letsencrypt:
  loki:
  ntfy:
  postgres:
  prometheus:
  promtail:
```

The only host bind mount is `/home/outsideworx/utils` (used by the `utils` container to write cached images to disk for the sites to serve).

## .env File (services)

All variables required for the services stack:

| Variable | Used by | Purpose |
|----------|---------|---------|
| `APP_CLIENTS_CIAFO_TOKEN` | services | API auth token for come-in-and-find-out |
| `APP_CLIENTS_PEEPS_TOKEN` | services | API auth token for gaiapeeps |
| `APP_CLIENTS_SOUP_TOKEN` | services | API auth token for soupart |
| `APP_CLIENTS_THEGREEN_SECRET` | sites (outsideworx) | Client secret for cookie-based access control (see `sites-wip.md`) |
| `AUTHELIA_IDENTITY_PROVIDERS_OIDC_HMAC_SECRET` | authelia | OIDC HMAC signing secret |
| `AUTHELIA_JWT_SECRET` | authelia | JWT signing secret |
| `AUTHELIA_SESSION_SECRET` | authelia | Session encryption secret |
| `AUTHELIA_STORAGE_ENCRYPTION_KEY` | authelia | Storage encryption key |
| `DISPATCH_TOKEN` | CI/CD | GitHub PAT (not used at runtime, kept for reference) |
| `GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET` | grafana | Grafana's OIDC client secret |
| `GF_SECURITY_ADMIN_PASSWORD` | grafana | Grafana admin password |
| `GF_SECURITY_ADMIN_USER` | grafana | Grafana admin username |
| `MAILERSEND_SDK_TOKEN` | services | MailerSend API token for email |
| `NTFY_ADMIN_TOKEN` | — | ntfy admin token (reference only) |
| `NTFY_WEB_PUSH_PRIVATE_KEY` | ntfy | Web push VAPID private key |
| `NTFY_WEB_PUSH_PUBLIC_KEY` | ntfy | Web push VAPID public key |
| `SPRING_DATASOURCE_PASSWORD` | services, postgres, postgres-exporter | PostgreSQL password |
| `SPRING_DATASOURCE_USERNAME` | services, postgres, postgres-exporter | PostgreSQL username (also DB name) |
| `SPRING_SECURITY_OAUTH2_CLIENT_REGISTRATION_AUTHELIA_CLIENT_SECRET` | services | OIDC client secret for Spring Security |

## Prod vs Test

| Aspect | Prod (compose.yaml + Swarm) | Test (compose-test.yaml + Compose) |
|--------|----------------------------|-------------------------------------|
| Orchestrator | Docker Swarm (`docker stack deploy`) | Docker Compose (`docker compose up`) |
| Network | External overlay `outsideworx` | Default compose network |
| Secrets | Docker secrets (external) | Not used (inline in config) |
| Placement | `constraints: node.role == manager` | Not applicable |
| Deploy mode | Swarm services with replicas | Plain containers |
| Volumes | Named Docker volumes + one host bind (`/home/outsideworx/utils`) | Ephemeral / local bind mounts |
| PostgreSQL auth | Username/password from `.env` | Trust auth (`POSTGRES_HOST_AUTH_METHOD: trust`) |
| Services image | `ghcr.io/outsideworx/services:latest` | Not in compose (runs on host via IDE) |
| Docker socket | `/var/run/docker.sock` | `/var/run/docker.sock` |

## CI/CD (GitHub Actions)

1. **Verify** — `mvn verify` on every push
2. **Build** — On successful verify of `main`: `mvn package -DskipTests`, Docker build + push to GHCR
3. **Deploy** — Manual trigger (`workflow_dispatch`): SSH to host, git pull, run `deploy.sh`

## File Layout

```
services/
├── .env                    # Environment variables (not committed in prod)
├── .github/workflows/
│   ├── verify.yaml         # mvn verify on push
│   ├── build.yaml          # Docker build on main
│   └── deploy.yaml         # SSH deploy (manual)
├── authelia.yaml           # Prod Authelia config
├── authelia-test.yaml      # Test Authelia config
├── authelia-users.yaml     # Prod user database
├── authelia-test-users.yaml # Test user database
├── compose.yaml            # Prod stack definition
├── compose-test.yaml       # Local dev stack
├── deploy.sh               # Deployment script
├── Dockerfile              # Spring Boot app image
├── grafana.ini             # Prod Grafana config
├── grafana-test.ini        # Test Grafana config
├── loki.yaml               # Prod Loki config
├── loki-test.yaml          # Test Loki config
├── logo.png                # Authelia branding
├── ntfy.yaml               # Prod ntfy config
├── ntfy-test.yaml          # Test ntfy config
├── pom.xml                 # Maven build
├── prometheus.yaml         # Prod Prometheus config
├── prometheus-test.yaml    # Test Prometheus config
├── promtail.yaml           # Prod Promtail config
├── promtail-test.yaml      # Test Promtail config
└── utils/                  # Python sidecar scripts
```
