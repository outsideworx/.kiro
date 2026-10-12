# wg-easy (Personal VPN)

## Overview

`wg-easy` is a WireGuard VPN server with a web admin UI, run for personal use. It is functionally independent of the rest of the platform — it has no relationship to any site or to the Spring Boot app/PostgreSQL — but it is fully integrated into the shared `services` stack infrastructure: it authenticates through Authelia (OIDC), is routed through Traefik, and is scraped by Prometheus, exactly like every other service.

- Image: `ghcr.io/wg-easy/wg-easy:15`
- Stack: `services` (defined alongside everything else in `services/compose.yaml`)
- Network: `edge` only (not on `outsideworx` — see `networking.md`)

## Container Configuration

```yaml
wg-easy:
  cap_add:
    - NET_ADMIN
  environment:
    DISABLE_VERSION_CHECK: "true"
    INIT_ENABLED: "true"
    OAUTH_AUTO_LAUNCH: "oidc"
    OAUTH_OIDC_CLIENT_ID: "wg-easy"
    OAUTH_OIDC_CLIENT_SECRET: $WG_OAUTH_OIDC_CLIENT_SECRET
    OAUTH_OIDC_SERVER: "https://oauth.outsideworx.net"
    OAUTH_PROVIDERS: "oidc"
  extra_hosts:
    - "oauth.outsideworx.net:host-gateway"
  image: ghcr.io/wg-easy/wg-easy:15
  networks:
    edge:
  ports:
    - mode: host
      protocol: udp
      published: 51820
      target: 51820
  sysctls:
    net.ipv4.conf.all.src_valid_mark: 1
    net.ipv4.ip_forward: 1
  volumes:
    - wireguard:/etc/wireguard
```

- **`cap_add: NET_ADMIN`** — required to create and manage the WireGuard network interface inside the container.
- **`sysctls`** — `ip_forward=1` lets the container route traffic between VPN peers and the outside; `src_valid_mark=1` is required for WireGuard's routing policy to work correctly inside a container's network namespace.
- **Port 51820/udp, `mode: host`** — the actual WireGuard tunnel traffic. Published directly on the host (bypassing the Swarm ingress mesh/VIP), the same way Traefik's 80/443 are. This is the only port that carries real VPN traffic; everything else about this service is just the admin UI.
- **Named volume `wireguard`** — persists the WireGuard server config and peer definitions at `/etc/wireguard`, declared under the stack's top-level `volumes:` alongside `cache`, `grafana`, etc.
- **`extra_hosts: oauth.outsideworx.net:host-gateway`** — lets the container resolve the public OIDC issuer hostname to the host, needed because `wg-easy` talks to Authelia using the external HTTPS URL (`OAUTH_OIDC_SERVER`) rather than an internal Swarm VIP/alias — unlike the Spring Boot app, which calls Authelia internally via the `services-authelia` network alias (see `networking.md`).

## OIDC Integration (Authelia)

`wg-easy` has its own Authelia OIDC client, alongside `grafana` and `services`:

```yaml
# authelia.yaml
- authorization_policy: one_factor
  consent_mode: implicit
  id: wg-easy
  redirect_uris:
    - https://vpn.outsideworx.net/api/auth/oidc/callback
    - https://vpn.outsideworx.net/api/auth/oidc/link
  scopes:
    - email
    - openid
    - profile
  secret: "<pbkdf2-sha512 hash>"
```

Differences from the other two clients: `wg-easy` requests **no `groups` scope** (it has no admin/viewer role mapping like Grafana does) and registers **two** redirect URIs — `/callback` (initial OIDC login) and `/link` (linking an OIDC identity to an existing local wg-easy account). `OAUTH_AUTO_LAUNCH: "oidc"` skips wg-easy's own login form and goes straight to the Authelia redirect.

The client secret is supplied via `WG_OAUTH_OIDC_CLIENT_SECRET` in `.env`, mapped to `OAUTH_OIDC_CLIENT_ID`/`OAUTH_OIDC_CLIENT_SECRET` in compose — the same pattern used for Grafana's and Spring Boot's OIDC client secrets. See `auth.md` for the full OIDC client table and the general Authelia setup.

## Traefik Integration

```yaml
# Prod — deploy.labels
- traefik.enable=true
- traefik.http.routers.wg-easy.entrypoints=websecure
- traefik.http.routers.wg-easy.rule=Host(`vpn.outsideworx.net`)
- traefik.http.routers.wg-easy.tls.certresolver=letsencrypt
- traefik.http.services.wg-easy.loadbalancer.server.port=51821
```

Traefik only routes the **admin UI** (port 51821, HTTP) under `vpn.outsideworx.net` — it has nothing to do with the actual WireGuard tunnel, which is the separately published UDP port 51820. This is why `wg-easy` needs a path to Traefik on the `edge` network rather than needing to be reachable from Traefik via `outsideworx`: the admin UI is just another web app behind the reverse proxy, same as Grafana or ntfy. See `traefik.md` for the full routing map and `networking.md` for why `edge` exists.

No `www-redirect` middleware (single-host rule, same reasoning as the npm app sites — there's only one real domain for this service).

## Prometheus Integration

```yaml
# prometheus.yaml
- job_name: wg-easy
  metrics_path: /metrics/prometheus
  static_configs:
    - targets: [ "services_wg-easy:51821" ]
  authorization:
    type: Bearer
    credentials_file: /run/secrets/wg_metrics_token
```

- wg-easy exposes real Prometheus metrics (not just a health-check string) at `/metrics/prometheus` on the same port as the admin UI (51821).
- The endpoint requires a bearer token — `wg-easy`'s own config, not an Authelia/Traefik concern. The token is a dedicated Docker secret, `wg_metrics_token`, read by Prometheus from the mounted secret file.
- Prometheus must be able to reach `wg-easy` directly (not through Traefik) on the `edge` network — this is one of the two reasons `prometheus` joins `edge` in addition to `outsideworx` (the other being wg-easy's own future use of Prometheus data, if any). See `networking.md`.
- In test (`prometheus-test.yaml`), the target is the short container name `wg-easy:51821` and the token is a hardcoded test credential rather than a secret file.

See `monitoring.md` for the full scrape target list and general Prometheus conventions.

## Network Placement

`wg-easy` joins **only** `edge` — it is not on `outsideworx` at all. It has no database, no API calls to the Spring Boot app, and no relationship to any site; its only two integration points (Traefik, Prometheus) are services that already straddle both networks. Putting it on `outsideworx` would gain it nothing and would be inconsistent with the "only join `outsideworx` if you need backend connectivity" rule used for the static sites. See `networking.md`'s "Why Prometheus and Traefik Are on `edge`" section — `wg-easy` is the reason that section exists.

## Secrets

| Secret/Variable | Where | Purpose |
|---|---|---|
| `wg_metrics_token` | Docker secret (external) | Bearer token Prometheus uses to scrape `/metrics/prometheus` |
| `WG_OAUTH_OIDC_CLIENT_SECRET` | `.env` → `OAUTH_OIDC_CLIENT_SECRET` env var | Authelia OIDC client secret for `wg-easy` |

### Generating `wg_metrics_token`

Added to `services/deploy.sh --secrets`, alongside the existing RSA key generation:

```bash
wg_metrics_token=$(openssl rand -base64 32)
echo "wg_metrics_token: $wg_metrics_token"
echo "$wg_metrics_token" | docker secret create wg_metrics_token -
```

The printed value must be copied into `prometheus.yaml`'s bearer credential (prod reads it from the secret file directly, so no manual copy is actually needed there — the print is for reference/rotation bookkeeping).

### Kernel Modules (`deploy.sh --network`)

wg-easy's NAT/forwarding needs iptables kernel modules not guaranteed to be loaded on a fresh host. `--network` now also writes a modules-load config and reloads it:

```bash
cat <<-EOF > /etc/modules-load.d/wg-easy.conf
    ip_tables
    iptable_nat
    ip6_tables
    ip6table_nat
EOF
systemctl restart systemd-modules-load.service
```

## Prod vs Test

| Aspect | Prod (`compose.yaml`) | Test (`compose-test.yaml`) |
|---|---|---|
| OIDC server | `https://oauth.outsideworx.net` | `https://oauth.localhost` |
| TLS trust | Normal validation | `NODE_TLS_REJECT_UNAUTHORIZED: 0` (self-signed Traefik cert in test) |
| OIDC client secret | `$WG_OAUTH_OIDC_CLIENT_SECRET` (from `.env`) | Hardcoded `wg-easy` |
| Metrics token | Docker secret (`wg_metrics_token`), file-mounted | Hardcoded test credential in `prometheus-test.yaml` |
| Domain | `vpn.outsideworx.net` | `vpn.localhost` |
| Volume | Named volume `wireguard` | Ephemeral (no volume declared) |
| TLS (Traefik) | `certresolver=letsencrypt` | `tls=true` (self-signed) |

## File Layout

No dedicated config file — all configuration is environment variables and compose labels, same as `postgres-exporter`. Nothing added to `services/`'s config-file list (`authelia.yaml`, `grafana.ini`, etc.) beyond the new Authelia client entry and the Prometheus scrape job, both inside existing files.

## Key Points

- **Not part of the sites/services business logic** — no API calls, no DB access, exists purely as a personal utility riding on the shared infrastructure.
- **Edge-only network placement** — never joins `outsideworx`.
- **Two distinct "surfaces"**: the raw WireGuard UDP tunnel (host-mode port 51820, outside Traefik entirely) and the admin/metrics HTTP UI (port 51821, routed by Traefik, scraped by Prometheus, protected by Authelia OIDC).
- **Reuses existing patterns rather than inventing new ones** — OIDC client the same shape as Grafana/services, Traefik labels the same shape as any other routed service, Prometheus bearer-token scraping is the only genuinely new pattern introduced (no other scrape target requires auth).
