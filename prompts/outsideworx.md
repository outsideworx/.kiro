# Outside Worx

At Outside Worx, we believe that every business deserves a digital presence as distinctive as its identity. Our team combines the expertise of a professional artist and a computer scientist to deliver websites that are both visually striking and technically refined. Guided by our lead designer's extensive artistic background, each project is crafted with attention to detail and a deep understanding of visual engagement. Every design is created to capture attention and communicate brand values with clarity and impact.

On the technical side, every site runs on infrastructure we built from scratch — isolated containers, encrypted connections, centralized logging, live analytics, and a custom admin portal with secure authentication. No prebuilt templates, no shared hosting, no third-party page builders.

AI-enabled from code to canvas — but never generic. We use AI to move faster and think bigger, while every pixel remains intentional and built for your needs. The result is something no template could produce: a site that is both AI-powered and entirely yours.

## Architecture

A self-hosted platform running multiple static websites and a shared backend on a single Docker Swarm node. Two stacks — **services** (backend + infrastructure) and **sites** (static websites) — share a single overlay network.

| Component | Stack | Description |
|-----------|-------|-------------|
| Apache Sites | sites | Single-Page web applications for clients |
| Authelia | services | OIDC identity provider for admin portal and Grafana |
| Monitoring | services | Prometheus, Grafana, Loki, Promtail, ntfy |
| PostgreSQL | services | Persistent data store for all client data |
| Spring Boot API | services | Java 25 backend with OAuth2 admin portal and token-based API auth |
| Traefik | services | Reverse proxy with automatic TLS via Let's Encrypt |

Each site is built from a shared Dockerfile into its own Apache httpd container. Sites that need dynamic content proxy API calls to the backend via internal networking. All sites get output rate limiting, request timeouts, IP blacklisting, and restrictive security headers out of the box. Work-in-progress sites can be protected with a lightweight cookie-based client secret without requiring full OAuth2.

For the full compose structure and deployment details, see the [services](https://github.com/outsideworx/.kiro/blob/main/prompts/outsideworx/services-deployment.md) and [sites](https://github.com/outsideworx/.kiro/blob/main/prompts/outsideworx/sites-deployment.md) deployment documentation.

## Documentation

Full platform documentation lives in the [`.kiro`](https://github.com/outsideworx/.kiro) repository — an AI-agent-readable knowledge base that also serves as human reference.

### System Prompts

| Document | Covers |
|----------|--------|
| [API Reference](https://github.com/outsideworx/.kiro/blob/main/prompts/outsideworx/api.md) | REST endpoint signatures, response shapes, pagination, caching, `/api/` vs `/api/cache/` |
| [Authentication](https://github.com/outsideworx/.kiro/blob/main/prompts/outsideworx/auth.md) | Authelia OIDC for admin portal, token-based API auth for sites |
| [GitHub Actions](https://github.com/outsideworx/.kiro/blob/main/prompts/outsideworx/github-actions.md) | CI/CD pipelines — verify, build, deploy workflows for both repos |
| [Monitoring](https://github.com/outsideworx/.kiro/blob/main/prompts/outsideworx/monitoring.md) | Prometheus, Grafana, Loki, Promtail, ntfy — metrics and log aggregation |
| [Networking](https://github.com/outsideworx/.kiro/blob/main/prompts/outsideworx/networking.md) | Overlay network, Swarm VIP DNS, hostname conventions, service communication graph |
| [Python Utilities](https://github.com/outsideworx/.kiro/blob/main/prompts/outsideworx/python-utils.md) | Sidecar scripts — image cache sync, operational shell scripts |
| [Services Deployment](https://github.com/outsideworx/.kiro/blob/main/prompts/outsideworx/services-deployment.md) | Docker Swarm stack for backend, PostgreSQL, monitoring, and supporting services |
| [Sites Deployment](https://github.com/outsideworx/.kiro/blob/main/prompts/outsideworx/sites-deployment.md) | Docker stack for Apache-based static sites, shared Dockerfile, proxy config |
| [Sites WIP](https://github.com/outsideworx/.kiro/blob/main/prompts/outsideworx/sites-wip.md) | Work-in-progress sites — submodule integration, client secret access control |
| [Spring Boot](https://github.com/outsideworx/.kiro/blob/main/prompts/outsideworx/spring-boot.md) | Java 25 / Spring Boot 3.5 backend — package structure, coding conventions, client pattern, testing |
| [Traefik](https://github.com/outsideworx/.kiro/blob/main/prompts/outsideworx/traefik.md) | Reverse proxy — TLS, routing, labels, middlewares |

### Skills (Step-by-Step Guides)

| Skill | Use when |
|-------|----------|
| [Admin Portal](https://github.com/outsideworx/.kiro/blob/main/skills/admin-portal/SKILL.md) | Creating or modifying client admin views (Thymeleaf templates, ModelVisitor controllers) |
| [Apache httpd](https://github.com/outsideworx/.kiro/blob/main/skills/httpd/SKILL.md) | Modifying the shared sites Dockerfile, security headers, proxy config, or rate limits |
| [Callback](https://github.com/outsideworx/.kiro/blob/main/skills/callback/SKILL.md) | Adding contact form callback to a client (email notification + DB persistence) |
| [GitHub](https://github.com/outsideworx/.kiro/blob/main/skills/github/SKILL.md) | Creating a new site repo, adding it to the build pipeline, or configuring repository settings |
| [Liquibase](https://github.com/outsideworx/.kiro/blob/main/skills/liquibase/SKILL.md) | Creating or modifying database tables, sequences, or triggers |
| [New Client](https://github.com/outsideworx/.kiro/blob/main/skills/new-client/SKILL.md) | Onboarding a new client — Authelia, API wiring, Spring Boot code, compose config |
| [New Site](https://github.com/outsideworx/.kiro/blob/main/skills/new-site/SKILL.md) | Creating the frontend repo for a new or existing client — file structure, page types, assets, scripts, CI |
| [SEO](https://github.com/outsideworx/.kiro/blob/main/skills/seo/SKILL.md) | Creating or modifying SEO files for a site — robots.txt, sitemap.xml, metrics.txt, meta tags |

### Steering

| Document | Purpose |
|----------|---------|
| [Coding Conventions](https://github.com/outsideworx/.kiro/blob/main/steering/coding-conventions.md) | Coding conventions — ordering rules, Java style, YAML structures, tests |
| [Ways of Working](https://github.com/outsideworx/.kiro/blob/main/steering/ways-of-working.md) | How the agent should behave — ask before acting, minimal responses, changelog management |
