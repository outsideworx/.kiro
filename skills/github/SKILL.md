---
name: github
description: GitHub repository and CI/CD setup. Use when creating a new site repo, adding a site to the build pipeline, or configuring repository settings.
---

# GitHub

## Overview

Procedural guide for GitHub-related tasks: creating repos, wiring dispatch workflows, registering sites in the build pipeline, and adding deploy workflows.

For the current state of all pipelines and their full configs, see the `github-actions` prompt.

## Repository Settings

### General

- Visibility: Public
- Default branch: `main`
- Template repository: disabled
- Archived / disabled: false

#### Features

| Feature | Value |
|---------|-------|
| Issues | disabled |
| Wiki | disabled |
| Projects | disabled |
| Discussions | disabled |
| Pages | disabled |
| Downloads (Releases) | enabled (default, leave as-is) |

#### Access

- `allow_forking`: true
- `web_commit_signoff_required`: true (enforced org-wide via org settings — no per-repo action needed)

#### Merge Options (Pull Requests)

| Option | Value |
|--------|-------|
| Allow merge commits | enabled |
| Allow squash merging | disabled |
| Allow rebase merging | disabled |
| Allow auto-merge | disabled |
| Always suggest updating pull request branches | disabled |
| Automatically delete head branches | disabled |

#### Danger Zone

- No transfer, archive, or visibility changes.

#### Other

| Option | Value |
|--------|-------|
| Allow comments on individual commits | disabled |
| Include Git LFS objects in archives | disabled |
| Limit how many branches and tags can be updated in a single push | disabled |

### Branch Protection (`main`)

All repos have identical branch protection on `main`:

#### Pull Requests

- Required approving reviews: 1
- Dismiss stale pull request approvals when new commits are pushed: enabled
- Require review from code owners: disabled
- Require approval of the most recent reviewable push: enabled
- Require conversation resolution before merging: disabled

#### Status Checks

- Require status checks to pass: enabled
- Require branches to be up to date before merging: enabled (strict)
- No specific status checks configured (checks are not required by name)

#### Other Rules

- Require signed commits: disabled
- Require linear history: disabled
- Require deployments to succeed: disabled
- Lock branch: disabled
- Do not allow bypassing the above settings (enforce admins): disabled

#### Push Access

- Restrict who can push: enabled
- No users, teams, or apps listed — effectively locks all direct pushes to `main`
- Force pushes: disabled
- Allow deletions: disabled
- Block force pushes: enabled

### Pages / Environments

- GitHub Pages: not configured
- Environments: not configured

## How to Create a New Site Repo

### 1. Create the Repository

Create at: `https://github.com/organizations/outsideworx/repositories/new`

Settings:
- Name: `<site-name>` (lowercase, hyphenated)
- Visibility: Public
- No template, no README (push initial commit directly)
- Default branch: `main`

### 2. Create the Dispatch Workflow

Create `.github/workflows/build.yaml`:

```yaml
name: Build
on:
  push:
    branches: [main]
jobs:
  dispatch:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.DISPATCH_TOKEN }}
          script: |
            await github.rest.repos.createDispatchEvent({
              owner: 'outsideworx',
              repo: 'sites',
              event_type: 'build-<site-name>',
              client_payload: { name: '<site-name>' }
            })
```

Replace `<site-name>` in both `event_type` and `name`.

### 3. Register in the Sites Build Pipeline

Delegate to the `new-site` skill — it owns compose, build matrix, and Prometheus wiring (see its "Infrastructure Wiring" section).

### 4. Verify

Push the initial commit to `main`. Confirm:
- The site repo's dispatch workflow runs
- The sites repo's `build` job triggers via `repository_dispatch`
- The image appears at `ghcr.io/outsideworx/<site-name>:latest`

## How to Add a Deploy Workflow

Deploy workflows follow a fixed template. Only `services` and `sites` repos have deploy workflows — individual site repos do not.

### Template

Create `.github/workflows/deploy.yaml`:

```yaml
name: Deploy
on:
  workflow_dispatch:
    inputs:
      flags:
        default: ''
        description: 'Flags to pass to deploy.sh'
        required: false
      host:
        default: 'services.outsideworx.net'
        description: 'SSH host to deploy to'
        required: true
jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
      - run: |
          set -e
          echo "${{ secrets.ENV }}" > .env
          ./deploy.sh "${{ github.event.inputs.flags }}"
```

### Required Secrets

All secrets are org-level and inherited automatically — no per-repo secret setup needed.

## How to Add a Verify Workflow (Java)

Only for repos with Maven builds:

```yaml
name: Verify
on:
  push:
jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '<current-version>'
      - run: mvn verify
```

## How to Add a Build Workflow (Docker Image)

For repos that produce a Docker image and need it pushed to GHCR after successful verification:

```yaml
name: Build
on:
  workflow_run:
    branches: [main]
    types: [completed]
    workflows: [Verify]
jobs:
  build:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '<current-version>'
      - run: mvn package -DskipTests
      - uses: docker/login-action@v3
        with:
          password: ${{ secrets.DISPATCH_TOKEN }}
          registry: ghcr.io
          username: ${{ github.actor }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/outsideworx/<repo-name>:latest
```

The `workflow_run` trigger chains this after `Verify` succeeds on `main`. Replace `<repo-name>` with the image name.

## How to Remove a Site from the Pipeline

1. Remove `build-<site-name>` from `repository_dispatch.types` in `sites/.github/workflows/build.yaml`
2. Remove `<site-name>` from the matrix in the `build-sites` job
3. Delete or archive the site repo on GitHub

## Conventions

- GHCR login uses `DISPATCH_TOKEN` as password and `github.actor` as username
- All images tagged `:latest` only (no semver, no SHA tags)
- Deploy is always manual — never auto-deploys
- YAML ordering rules are in the `coding-conventions` steering doc
