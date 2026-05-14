# Outside Worx — Agent Overview

## Role

This agent manages the Outside Worx infrastructure: a self-hosted platform running multiple static websites and a shared backend on a single Docker Swarm node.

## Context Files

The following prompt files provide detailed documentation for each subsystem:

| File | Covers |
|------|--------|
| `spring-boot.md` | The Java 25 / Spring Boot 3.5 backend — package structure, coding conventions, client pattern, testing |
| `auth.md` | Authentication: Authelia OIDC for the admin portal, token-based API auth for sites |
| `services-deployment.md` | Docker Swarm stack for the backend, PostgreSQL, monitoring, and supporting services |
| `sites-deployment.md` | Docker stack for the Apache-based static sites, shared Dockerfile, proxy config |
| `traefik.md` | Traefik v3 reverse proxy — TLS, routing, labels, middlewares |
| `monitoring.md` | Prometheus, Grafana, Loki, Promtail, ntfy — metrics and log aggregation |
| `networking.md` | Overlay network, Swarm VIP DNS, network aliases, hostname conventions, service communication graph |
| `github-actions.md` | CI/CD pipelines — verify, build, deploy workflows for both repos |
| `python-utils.md` | Sidecar scripts — image cache sync, operational shell scripts |
| `new-user.md` | Procedure for onboarding a new user across Authelia, ntfy, and the Spring Boot portal |

## Steering

| File | Purpose |
|------|---------|
| `steering/conventions.md` | Coding conventions — Ordering rules, Java style, YAML structures, tests |
| `steering/notes.md` | Persistent instructions (e.g., how to handle "remember this") |
| `steering/changelog.md` | Running log of decisions and changes across sessions |

## Skills (on-demand)

| File | Use when |
|------|----------|
| `skills/liquibase/SKILL.md` | Creating or modifying database tables, sequences, or triggers |
| `skills/httpd/SKILL.md` | Modifying the shared sites Dockerfile, security headers, proxy config, or rate limits |

## Key Facts

- Single server, single Swarm manager node
- Two Docker stacks: `services` (backend + infra) and `sites` (static sites)
- Shared overlay network: `outsideworx`
- All images pushed to GHCR (`ghcr.io/outsideworx/<name>:latest`)
- Deployment is always manual (`workflow_dispatch`)
- Prod domains are real TLDs; test uses `*.localhost`
- Several deployed sites, plus work-in-progress sites hosted as submodules of the `outsideworx` repo (served under `outsideworx.net/clients/<name>/`)
- Some sites have API tokens (e.g., `come-in-and-find-out`, `gaiapeeps`, `soupart`)
- WIP sites (e.g., `thegreen`) are separate repos added as git submodules — they share the outsideworx container and don't get their own compose entry until they graduate to a standalone domain
- Auth: Authelia OIDC for admin portal, static token headers (`X-Auth-Token` + `X-Caller-Id`) for API
- Monitoring: Prometheus scrapes all services, Loki/Promtail for logs, ntfy for notifications, Grafana for dashboards
- All config has prod/test variants; test uses Docker Compose (not Swarm), trust auth, permissive CORS, `*.localhost` domains
- Deploy host: `services.outsideworx.net` (both stacks deploy to the same server)

