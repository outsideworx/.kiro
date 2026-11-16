# Documentation Map

Which document owns which kind of fact, and how to keep them in sync. When two documents disagree, the **code and config files win**; then fix the document that owns the fact and make the others link to it instead of repeating it.

## Layers

| Layer | Lives in | Owns |
|-------|----------|------|
| Platform prompts | `.kiro/prompts/outsideworx/*.md` | Cross-repo behaviour: how the stacks, networks, CI, auth and monitoring fit together |
| Repo guides | `<repo>/AGENTS.md`, `<repo>/.agents/summary/*.md` | One repo's layout, patterns and gotchas. Not under version control yet; read the repo's `AGENTS.md` before editing in it |
| Skills | `.kiro/skills/*/SKILL.md` | Step-by-step procedures (add a site, add a client, change Apache, write a changelog) |
| Steering | `.kiro/steering/*.md` | Coding conventions and how the agent should behave |

## Owner per fact

| Fact | Owner |
|------|-------|
| Site list, domains, site types, token and cache-volume use | `sites-deployment.md` (sites table) |
| Which service joins which network, hostname conventions | `networking.md` |
| Traefik routing, middlewares, labels | `traefik.md` |
| Prometheus targets, Loki/Promtail/Grafana/ntfy settings | `monitoring.md` |
| Build matrix, dispatch payloads, workflow behaviour, `DOC:` rule | `github-actions.md` |
| Compose structure, `deploy.sh`, configs, secrets, `.env` variables of the services stack | `services-deployment.md` |
| Authelia, OIDC clients, API token model, CORS | `auth.md` |
| Endpoint shapes, pagination, response fields | `api.md` |
| Spring Boot package layout and client pattern | `spring-boot.md` |
| wg-easy, WIP/secret-gated sites, Python sidecar | `wg-easy.md`, `sites-wip.md`, `python-utils.md` |
| Failure symptoms and fixes | `troubleshooting.md` |
| Registering a site in compose/matrix/Prometheus (procedure) | `new-site` skill, `references/infra-wiring.md` |
| Ordering and style rules | `steering/coding-conventions.md` |
| Safety rules (no mutating Docker commands) | `steering/ways-of-working.md` |

## Keeping it in sync

- Change a fact in its owner first; elsewhere write a pointer ("see `networking.md`"), not a copy. Avoid counts ("8 sites") — list names or say "every site".
- After adding or removing a site, run `python3 .kiro/skills/new-site/scripts/check_sites.py <workspace-root>` (matrix, both compose files, Prometheus targets).
- After editing a changelog, run `python3 .kiro/skills/liquibase/scripts/check_changelog.py <db-dir>`; for a site's SEO files, `python3 .kiro/skills/seo/scripts/check_seo.py <site-dir>`.
- After editing any doc, run `python3 .kiro/scripts/check_stale_docs.py <workspace-root>`; it flags phrases already known to be wrong (e.g. "Apache + Node" for the npm image, "only if the pointer moved"). Add a rule when a fix shows a claim that could creep back.
- Some claims cannot be checked from the repos (GitHub branch protection, runner tooling, the wg-easy admin UI). Those sections carry a "not verifiable from the repo" note; confirm them in the live system before relying on a detail.
