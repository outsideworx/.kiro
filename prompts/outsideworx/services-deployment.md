# Services Deployment

## Overview

The services stack runs on Docker Swarm as a single-node cluster. It contains the Spring Boot app, PostgreSQL, monitoring, auth, and utility containers — on two shared overlay networks (`outsideworx` and `edge`; see `networking.md`).

## Infrastructure

- Single manager node (all constraints: `node.role == manager`)
- Overlay networks: `outsideworx` and `edge` (both external, attachable — see `networking.md` for which services join which)
- Docker secrets: all credentials of the stack (see "Docker Secrets (services)")
- Stack name: `services`

## deploy.sh

The deployment script handles both initial setup and updates.

### Flags

| Flag | Action |
|------|--------|
| `--install` | Installs `docker-compose-v2`, `git`, `maven`, `npm`, `openjdk-25-jdk` via apt |
| `--network` | Initializes Swarm (`docker swarm init --advertise-addr <IP>`) and creates both overlay networks (`edge`, `outsideworx`) |
| `--secrets` | Creates all Docker secrets (generates RSA key + random `openssl rand -base64 32` values and prints them; prompts for `mailersend_sdk_token` and the two OIDC client secrets); skips existing ones |
| (no flag) | Deploys/updates the stack |

### Deployment Flow (no flag)

1. Creates `/home/outsideworx/services/` (if missing)
2. Copies `utils/` directory and config files to deploy directory
3. Sources `.env` file
4. Computes SHA256 hashes of config files (for Swarm config naming)
5. `docker compose pull`
6. `docker stack deploy -c compose.yaml services --detach=false --resolve-image=always`
7. Force-updates all services (rolling restart)

### Prerequisites

- Docker Swarm initialized (`--network` flag, requires IP address)
- Both overlay networks created (`--network` flag)
- Docker secrets created (`--secrets` flag, interactive, run on the host)
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
| wg-easy | ghcr.io/wg-easy/wg-easy | via Traefik (admin UI) + host-mode UDP 51820 (VPN) | See the `wg-easy` prompt |

### Network

Most services join the `outsideworx` external overlay network. `traefik` and `prometheus` join both `outsideworx` and `edge`. No other service-specific networks. See `networking.md` for the full per-service breakdown and rationale.

### Secrets

All secrets are declared `external: true` and created by `deploy.sh --secrets`. See "Docker Secrets (services)" below for the full list and how each consumer reads them.

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
  cache:
  grafana:
  letsencrypt:
  loki:
  ntfy:
  postgres:
  prometheus:
  promtail:
  wireguard:
```

The `cache` named volume holds the images the `utils` container syncs from PostgreSQL. It is exposed to the sites stack as the external volume `services_cache` (Swarm prefixes the stack name), which `come-in-and-find-out` and `soupart` mount read-only. This replaced the former `/home/outsideworx/utils` host bind mount — `deploy.sh` no longer creates that directory. The `wireguard` volume holds wg-easy's config/peers (see the `wg-easy` prompt).

Host bind mounts:
- `/var/run/docker.sock` → traefik (service discovery) and promtail (container log discovery)

## Docker Secrets (services)

All external, created once by `deploy.sh --secrets` (idempotent: existing secrets are skipped; `rsa_private_key` is generated with `openssl genrsa 4096`, `grafana_oauth_client_secret`, `mailersend_sdk_token` and `services_oauth_client_secret` are prompted for — the OIDC client secrets must match the pbkdf2 hashes committed in `authelia.yaml`, and the MailerSend token is issued externally — all others are generated with `openssl rand -base64 32` and printed so they can be entered where needed after install). Values are written with `printf '%s'` (no trailing newline). Swarm secrets are immutable — to rotate, `docker secret rm` (after removing consumers) and re-run `--secrets`.

| Secret | Consumed by | Mechanism |
|--------|-------------|-----------|
| `app_clients_ciafo_token` | services, sites (come-in-and-find-out) | configtree → `app.clients.ciafo.token`; sites: `/run/secrets/token` |
| `app_clients_peeps_token` | services, sites (gaiapeeps) | configtree → `app.clients.peeps.token`; sites: `/run/secrets/token` |
| `app_clients_soup_token` | services, sites (soupart) | configtree → `app.clients.soup.token`; sites: `/run/secrets/token` |
| `app_clients_worx_token` | services, sites (outsideworx) | configtree → `app.clients.worx.token`; sites: `/run/secrets/token` |
| `authelia_jwt_secret` | authelia | `AUTHELIA_JWT_SECRET_FILE` |
| `authelia_oidc_hmac_secret` | authelia | `AUTHELIA_IDENTITY_PROVIDERS_OIDC_HMAC_SECRET_FILE` |
| `authelia_session_secret` | authelia | `AUTHELIA_SESSION_SECRET_FILE` |
| `authelia_storage_encryption_key` | authelia | `AUTHELIA_STORAGE_ENCRYPTION_KEY_FILE` |
| `grafana_oauth_client_secret` | grafana | `$__file{...}` in `grafana.ini` |
| `mailersend_sdk_token` | services | configtree → `mailersend.sdk.token` |
| `postgres_password` | postgres, postgres-exporter, services, utils | `POSTGRES_PASSWORD_FILE`, `DATA_SOURCE_PASS_FILE`, configtree → `spring.datasource.password`, `DB_PASSWORD_FILE` |
| `rsa_private_key` | authelia | `AUTHELIA_IDENTITY_PROVIDERS_OIDC_ISSUER_PRIVATE_KEY_FILE` |
| `services_oauth_client_secret` | services | configtree → `spring.security.oauth2.client.registration.authelia.client-secret` |
| `wg_metrics_token` | prometheus | `credentials_file` |

The `services` container sets `SPRING_CONFIG_IMPORT: configtree:/run/secrets/`; each secret's `target` is the Spring property path. Grafana has no admin user/password configured: the login form and basic auth are disabled, access is OAuth only.

## .env File (services)

Remaining variables (non-secret, or not yet migrated because the image has no file-based input):

| Variable | Used by | Purpose |
|----------|---------|---------|
| `APP_CLIENTS_THEGREEN_SECRET` | sites (outsideworx) | Client secret for cookie-based access control (see `sites-wip.md`) |
| `DISPATCH_TOKEN` | CI/CD | GitHub PAT (not used at runtime, kept for reference) |
| `NTFY_ADMIN_TOKEN` | — | ntfy admin token (reference only) |
| `NTFY_WEB_PUSH_PRIVATE_KEY` | ntfy | Web push VAPID private key (not migrated: no file input) |
| `NTFY_WEB_PUSH_PUBLIC_KEY` | ntfy | Web push VAPID public key |
| `SPRING_DATASOURCE_USERNAME` | services, postgres, postgres-exporter, utils | PostgreSQL username (also DB name) |
| `WG_OAUTH_OIDC_CLIENT_SECRET` | wg-easy | Authelia OIDC client secret for wg-easy (not migrated: no file input; see the `wg-easy` prompt) |

The file also carries variables consumed outside the services stack: `TUNDE_DIVAT_*` (consumed by the `tunde-divat` app — `apps/api/src/config/env.ts`, `prisma/seed.ts` — mapped to its unprefixed env names in `sites/compose.yaml`). See `sites-deployment.md`.

## Prod vs Test

| Aspect | Prod (compose.yaml + Swarm) | Test (compose-test.yaml + Compose) |
|--------|----------------------------|-------------------------------------|
| Orchestrator | Docker Swarm (`docker stack deploy`) | Docker Compose (`docker compose up`) |
| Network | External overlays `outsideworx` + `edge` | Default compose network |
| Secrets | Docker secrets (external) | Not used (inline in config) |
| Placement | `constraints: node.role == manager` | Not applicable |
| Deploy mode | Swarm services with replicas | Plain containers |
| Volumes | Named Docker volumes + host bind mount (`/var/run/docker.sock`) | Ephemeral / local bind mounts |
| PostgreSQL auth | Username from `.env`, password from Docker secret `postgres_password` | Trust auth (`POSTGRES_HOST_AUTH_METHOD: trust`) |
| Services image | `ghcr.io/outsideworx/services:latest` | Not in compose (runs on host via IDE) |
| Docker socket | `/var/run/docker.sock` | `/var/run/docker.sock` |

## CI/CD (GitHub Actions)

1. **Verify** — `mvn verify` on every push
2. **Build** — On successful verify of `main`: `mvn package -DskipTests`, Docker build + push to GHCR
3. **Deploy** — Manual trigger (`workflow_dispatch`): checks out repo, writes `.env` from secret, runs `deploy.sh` directly on the host

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
