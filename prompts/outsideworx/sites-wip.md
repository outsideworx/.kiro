# Work-in-Progress Sites

## Overview

WIP sites are client projects not yet ready for their own domain. They live as git submodules of an existing site repo and are protected by a lightweight cookie-based secret. When ready, they graduate to a standalone compose service with their own domain.

## Submodule Integration

WIP sites are separate repos added as git submodules of the `outsideworx` repo. They share the `outsideworx` container — no separate compose entry.

### How It Works

```
outsideworx repo
├── .gitmodules              # Declares submodules
├── clients/
│   └── thegreen/            # Submodule → github.com/outsideworx/thegreen
└── index.html
```

At build time, the `fetcher` stage clones the outsideworx repo and initializes submodules:

```dockerfile
RUN git clone --depth 1 https://github.com/outsideworx/outsideworx.git /sites
RUN git -C /sites submodule update --init --depth 1
```

The submodule content ends up at `/usr/local/apache2/htdocs/clients/thegreen/` inside the container. Apache's `MultiViews` and `DirectoryIndex` directives on `htdocs/clients/` enable extensionless URL serving.

The site is accessible at `outsideworx.net/clients/thegreen/`.

### Current WIP Sites

| Name | Submodule Path | Protected Path | Repo |
|------|----------------|----------------|------|
| thegreen | `clients/thegreen/` | `/clients/thegreen/` | `outsideworx/thegreen` |

### Updating Submodule Content

When the WIP site repo is updated, the outsideworx repo's submodule pointer must be updated manually:

```bash
cd outsideworx
git submodule update --remote clients/thegreen
git add clients/thegreen
git commit -m "Update thegreen submodule"
git push
```

This triggers the outsideworx site rebuild via `repository_dispatch`.

### Graduating to Standalone

When a WIP site is ready for its own domain:

1. Remove the submodule from the outsideworx repo (`git rm clients/<name>`)
2. Add a new service entry in `sites/compose.yaml` and `sites/compose-test.yaml`
3. Add Traefik labels with the real domain
4. Add to the build matrix and dispatch types in `sites/.github/workflows/build.yaml`
5. Add to Prometheus scrape targets
6. Remove the `CLIENT_SECRET` environment variables (no longer needed)
7. Optionally follow the `new-client` skill if it also needs API access

## Client Secret Access Control

A lightweight protection mechanism for WIP sites that need to share demo URLs with clients without exposing unfinished content to the public. Avoids the overhead of full OAuth2 — just "enter secret to view."

### How It Works

```
Request to ${CLIENT_SECRET_PATH}page
        │
        ▼
LuaHookAccessChecker (secret.lua)
        │
        ├── Has cookie access_token=granted? → Pass through (DECLINED)
        │
        └── No cookie → 302 redirect to ${CLIENT_SECRET_PATH}secret
                                │
                                ▼
                        Renders HTML form (dark-themed, password input)
                                │
                                ▼ POST
                        Compare submitted password to CLIENT_SECRET
                                │
                        ├── Match → Set cookie, 302 to protected path
                        └── Mismatch → Re-render form with error
```

The template (`secret.lua.tpl`) has two placeholders: `{{CLIENT_SECRET}}` and `{{CLIENT_SECRET_PATH}}`.

- **Cookie attributes**: `HttpOnly` (no JS access), `SameSite=Strict` (no CSRF), `Path={CLIENT_SECRET_PATH}` (scoped to protected path)
- **No `Secure` flag**: works on HTTP in test mode
- **Session cookie**: no `Max-Age` or `Expires` — cleared when browser closes
- **No server-side state**: the cookie itself is the proof of access

At container startup, the entrypoint script (`docker-entrypoint.sh`) checks for `CLIENT_SECRET`:

1. If set: runs `sed` on `secret.lua.tpl` to inject `CLIENT_SECRET` and `CLIENT_SECRET_PATH`, producing `/usr/local/apache2/conf/secret.lua`
2. Generates `conf/extra/httpd-auth.conf` with:
   ```apache
   <Location "${CLIENT_SECRET_PATH}">
       LuaHookAccessChecker /usr/local/apache2/conf/secret.lua check_access
   </Location>
   ```
3. If not set: writes `# No auth` to `httpd-auth.conf` (no-op include)

Requires `mod_lua` (enabled in both Dockerfiles via `sed`).

### Configuration

Compose environment variables on the site service:

```yaml
environment:
  CLIENT_SECRET: $APP_CLIENTS_<CLIENT>_SECRET
  CLIENT_SECRET_PATH: "/path/to/protect/"
```

The path **must** end with `/`. All requests under that path require the secret.

### Adding Client Secret to Another Site

1. Add `CLIENT_SECRET` and `CLIENT_SECRET_PATH` to the site's environment in `compose.yaml`
2. Add the same in `compose-test.yaml` (hardcoded value for local dev)
3. Add `APP_CLIENTS_<CLIENT>_SECRET=<value>` to both `sites/.env` and `services/.env`
4. No Dockerfile changes needed — the entrypoint handles it automatically

### Prod vs Test

| Aspect | Prod | Test |
|--------|------|------|
| Secret value | From `.env` via `$APP_CLIENTS_<CLIENT>_SECRET` | Hardcoded in `compose-test.yaml` |
| Cookie `Secure` flag | Not set (works over both HTTP and HTTPS) | Not set |
| Path | Same in both | Same in both |
