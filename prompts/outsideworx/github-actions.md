# GitHub Actions CI/CD

## Overview

Both repos (`outsideworx/services` and `outsideworx/sites`) use GitHub Actions for CI/CD. Images are pushed to GitHub Container Registry (GHCR). Deployment is SSH-based and manually triggered.

## Services Pipeline

```
Push (any branch) → Verify → (main only, on success) → Build → (manual) → Deploy
```

### Verify (`verify.yaml`)

- Trigger: every push (all branches)
- Steps: checkout → setup Java 25 (Temurin) → `mvn verify`
- Runs unit + integration tests (Testcontainers)

### Build (`build.yaml`)

- Trigger: `workflow_run` — runs after Verify completes successfully on `main`
- Condition: `github.event.workflow_run.conclusion == 'success'`
- Steps: checkout → setup Java 25 → `mvn package -DskipTests` → Docker login → build + push
- Image: `ghcr.io/outsideworx/services:latest`
- Tag strategy: always `latest` (no versioned tags)

### Deploy (`deploy.yaml`)

- Trigger: `workflow_dispatch` (manual)
- Inputs:
  - `host` — SSH target (default: `services.outsideworx.net`)
  - `flags` — passed to `deploy.sh` (e.g., `--network`, `--secrets`)
- Steps: SSH to host → clone repo if missing → write `.env` if missing → git pull → `bash deploy.sh <flags>`

## Sites Pipeline

```
Push to main → Build all sites (matrix)
Site repo update → repository_dispatch → Build single site
(manual) → Deploy
```

### Build (`build.yaml`)

Two jobs, mutually exclusive:

#### `build-sites` (push to main)

- Trigger: push to `main` branch
- Strategy: matrix over all 7 site names
- Steps: checkout → Docker login → build + push with `NAME` build arg
- Images: `ghcr.io/outsideworx/<name>:latest`

#### `build` (repository_dispatch)

- Trigger: `repository_dispatch` from individual site repos
- Event types: `build-come-in-and-find-out`, `build-duckumbrella`, `build-gaiapeeps`, `build-igli`, `build-outsideworx`, `build-soupart`, `build-soupkitchen`
- Payload: `{ "name": "<site-name>" }` in `client_payload`
- Builds only the dispatched site

### Deploy (`deploy.yaml`)

- Trigger: `workflow_dispatch` (manual)
- Inputs: `host`, `flags` (same pattern as services)
- Steps: SSH → clone if missing → write `.env` if missing → git pull → `bash deploy.sh <flags>`

## Cross-Repo Dispatch Pattern

Individual site repos trigger a rebuild of their image in the sites repo:

```yaml
# In a site repo's workflow:
- uses: peter-evans/repository-dispatch@v3
  with:
    token: ${{ secrets.DISPATCH_TOKEN }}
    repository: outsideworx/sites
    event-type: build-<name>
    client-payload: '{"name": "<name>"}'
```

This allows site content changes to trigger a Docker rebuild without touching the sites repo.

## Secrets

| Secret | Used in | Purpose |
|--------|---------|---------|
| `DISPATCH_TOKEN` | services, sites | GitHub PAT — GHCR login + cross-repo dispatch |
| `SSH_USER` | services, sites | SSH username for deploy |
| `SSH_PRIVATE_KEY` | services, sites | SSH private key for deploy |
| `ENV` | services, sites | Full `.env` file content (written on first deploy) |

## Actions Used

| Action | Version | Purpose |
|--------|---------|---------|
| `actions/checkout` | v4 | Clone repo |
| `actions/setup-java` | v4 | Install Temurin JDK 25 |
| `docker/login-action` | v3 | Authenticate to GHCR |
| `docker/build-push-action` | v6 | Build and push Docker image |
| `appleboy/ssh-action` | v1 | Execute deploy script via SSH |

## Conventions

- All images tagged `:latest` only (no semver, no SHA tags)
- GHCR registry: `ghcr.io/outsideworx/<name>`
- Runner: `ubuntu-latest` for all jobs
- Deploy is always manual (`workflow_dispatch`) — never auto-deploys
- The `.env` file is created from the `ENV` secret only on first deploy (subsequent deploys use the existing file)
- Deploy host defaults to `services.outsideworx.net` (same server hosts both stacks)

## Adding a New Site to CI

1. Add `build-<name>` to the `repository_dispatch.types` list in `sites/.github/workflows/build.yaml`
2. Add `<name>` to the matrix in the `build-sites` job
3. In the new site's own repo, add a workflow that dispatches `build-<name>` to `outsideworx/sites`
