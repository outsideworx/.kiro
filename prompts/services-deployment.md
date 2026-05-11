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

1. Wipes and recreates `/home/outsideworx/services/`
2. Creates persistent data directories:
   - `/home/outsideworx/data` (PostgreSQL)
   - `/home/outsideworx/grafana`
   - `/home/outsideworx/letsencrypt`
   - `/home/outsideworx/loki`
   - `/home/outsideworx/ntfy`
   - `/home/outsideworx/prometheus`
   - `/home/outsideworx/promtail`
   - `/home/outsideworx/utils`
3. Copies `utils/` directory and config files to deploy directory
4. Sources `.env` file
5. `docker compose pull`
6. `docker stack deploy -c compose.yaml services --detach=false --resolve-image=always`
7. Force-updates all services (rolling restart)

### Prerequisites

- Docker Swarm initialized (`--network` flag, requires IP address)
- Overlay network created (`--network` flag)
- RSA secret created (`--secrets` flag)
- `.env` file present in repo root with all required variables

## Compose Structure (compose.yaml)

### Services

| Service | Image | Exposed | Notes |
|---------|-------|---------|-------|
| traefik | traefik:v3.6.14 | 80, 443 (host mode) | See traefik.md |
| authelia | authelia/authelia:4.37 | via Traefik | See auth.md |
| services | ghcr.io/outsideworx/services:latest | via Traefik | Spring Boot app |
| postgres | postgres:16.2 | internal only | Data at `/home/outsideworx/data` |
| grafana | grafana/grafana:12.3.0 | via Traefik | See monitoring.md |
| loki | grafana/loki:3.6.4 | internal only | See monitoring.md |
| prometheus | prom/prometheus:v3.9.1 | internal only | See monitoring.md |
| promtail | grafana/promtail:3.6.4 | internal only | Global deploy mode |
| ntfy | binwiederhier/ntfy:v2.16.0 | via Traefik | See monitoring.md |
| postgres-exporter | prometheuscommunity/postgres-exporter:v0.18.1 | internal only | |
| utils | python:3.13-slim | internal only | See python-utils.md |

### Network

All services join the `outsideworx` external overlay network. No service-specific networks.

### Secrets

```yaml
secrets:
  rsa_private_key:
    external: true
```

Used only by the `authelia` service.

## .env File (services)

All variables required for the services stack:

| Variable | Used by | Purpose |
|----------|---------|---------|
| `APP_CLIENTS_CIAFO_TOKEN` | services | API auth token for come-in-and-find-out |
| `APP_CLIENTS_PEEPS_TOKEN` | services | API auth token for gaiapeeps |
| `APP_CLIENTS_SOUP_TOKEN` | services | API auth token for soupart |
| `AUTHELIA_IDENTITY_PROVIDERS_OIDC_HMAC_SECRET` | authelia | OIDC HMAC signing secret |
| `AUTHELIA_JWT_SECRET` | authelia | JWT signing secret |
| `AUTHELIA_SESSION_SECRET` | authelia | Session encryption secret |
| `AUTHELIA_STORAGE_ENCRYPTION_KEY` | authelia | Storage encryption key |
| `GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET` | grafana | Grafana's OIDC client secret |
| `GF_SECURITY_ADMIN_PASSWORD` | grafana | Grafana admin password |
| `GF_SECURITY_ADMIN_USER` | grafana | Grafana admin username |
| `GITHUB_TOKEN` | CI/CD | GitHub PAT (not used at runtime, kept for reference) |
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
| Volumes | Persistent host paths (`/home/outsideworx/...`) | Ephemeral / local bind mounts |
| PostgreSQL auth | Username/password from `.env` | Trust auth (`POSTGRES_HOST_AUTH_METHOD: trust`) |
| Services image | `ghcr.io/outsideworx/services:latest` | Not in compose (runs on host via IDE) |
| Docker socket | `/var/run/docker.sock` | `$HOME/.docker/run/docker.sock` (macOS) |

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
├── compose.yaml            # Prod stack definition
├── compose-test.yaml       # Local dev stack
├── deploy.sh               # Deployment script
├── Dockerfile              # Spring Boot app image
└── pom.xml                 # Maven build
```
