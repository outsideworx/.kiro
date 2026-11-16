---
name: httpd
description: Apache httpd conventions for the shared sites Dockerfile: proxy to services, response headers (CSP, Permissions-Policy, X-Frame-Options), RemoteIP/Traefik trust, rate limits, output limits and timeouts, RedirectMatch 403 blocks, redirects, modules, entrypoint (TOKEN). Use whenever the user wants to change Apache behaviour for the sites — add a header, loosen the CSP for fonts or a CDN, block a file type like .bak, raise a rate limit or timeout, add a redirect, enable a module — even if they never say 'httpd'.
---

# Apache httpd — Sites Configuration

## Overview

All sites share a single `Dockerfile` (prod) and `Dockerfile.test` (test). Apache config is generated inline via `RUN cat <<'EOF'` blocks — there are no external `.conf` files checked into the repo (except `blacklist.conf` and `blacklist-test.conf`).

For the current configuration values (modules, headers, rate limits, timeouts, URL blocking, redirects, prod vs test differences), see `sites-deployment.md`.

## Config File Structure

Includes appended to `httpd.conf` via `sed -e '$aInclude ...'`:

| File | Purpose |
|------|---------|
| `conf/extra/blacklist.conf` (prod) / `conf/extra/blacklist-test.conf` (test) | IP deny rules (prod: real IPs, test: placeholder) |
| `conf/extra/httpd-auth.conf` | Client secret Lua auth (created by the entrypoint; `# No auth` when unset) |
| `conf/extra/httpd-logs.conf` | Log format and output |
| `conf/extra/httpd-token.conf` | TOKEN env var injection (prod only, created by entrypoint) |
| `conf/extra/httpd-proxy.conf` | Proxy, MPM, rate limit, headers, redirects, directory options |
| `conf/extra/httpd-remoteip.conf` | RemoteIP trusted proxy ranges |

## Entrypoint

```sh
#!/bin/sh
echo "Define TOKEN ${TOKEN:-none}" > /usr/local/apache2/conf/extra/httpd-token.conf
if [ -n "${CLIENT_SECRET}" ]; then
    sed -e "s|{{CLIENT_SECRET}}|${CLIENT_SECRET}|g" \
        -e "s|{{CLIENT_SECRET_PATH}}|${CLIENT_SECRET_PATH:-/}|g" \
        /usr/local/apache2/conf/secret.lua.tpl > /usr/local/apache2/conf/secret.lua
    cat <<AUTHCONF > /usr/local/apache2/conf/extra/httpd-auth.conf
<Location "${CLIENT_SECRET_PATH:-/}">
    LuaHookAccessChecker /usr/local/apache2/conf/secret.lua check_access
</Location>
AUTHCONF
else
    echo "# No auth" > /usr/local/apache2/conf/extra/httpd-auth.conf
fi
exec httpd-foreground
```

- `TOKEN` is always written (defaults to `none` if unset) — uses Apache `Define` directive
- Client secret auth is conditionally configured (see `sites-wip.md`)
- The test Dockerfile's entrypoint is identical except it does not write `httpd-token.conf` (the proxy conf hardcodes `"test"`); the client-secret Lua logic and `httpd-auth.conf` exist in both

## Enabling a Module

Modules are uncommented via `sed` in the Dockerfile:

```dockerfile
RUN sed -i \
    -e 's|#LoadModule headers_module modules/mod_headers.so|LoadModule headers_module modules/mod_headers.so|' \
    -e 's|#LoadModule lua_module modules/mod_lua.so|LoadModule lua_module modules/mod_lua.so|' \
    conf/httpd.conf
```

To add a new module, append another `-e` line to this `sed` block (alphabetical order).

## Proxy Target

Prod and test differ only in the `ProxyPass`/`ProxyPassReverse` target (`http://services_services/api/` vs `http://host.docker.internal:8080/api/`); see `sites-deployment.md` for the current values. Change both Dockerfiles together.

## Response Headers

Headers are `Header always set <Name> "<value>"` lines in `httpd-proxy.conf`. To add one, append a line there (alphabetical by header name after `X-Request-Id`) in both Dockerfiles. The current list is in `sites-deployment.md`.

### Content-Security-Policy

Before loosening anything, check the table below: `font-src`, `frame-src`, `media-src` already allow any `https:` origin and `script-src`/`style-src` allow any origin, so a blocked font or embed from an https CDN usually has another cause (`data:` fonts, `connect-src 'self'` for fetch/XHR, a missing CORS header on the CDN, or a stale image). Ask for the console error before editing. Edit both `Dockerfile` and `Dockerfile.test`.

```
base-uri        'none'
connect-src     'self'
default-src     'none'
font-src         *       https:
frame-ancestors 'none'
frame-src        *       https:
form-action     'self'
img-src         'self'   data:
media-src        *       https:
script-src       *      'unsafe-inline'
style-src        *      'unsafe-inline'
```

## RemoteIP (Traefik trust)

```apache
RemoteIPHeader X-Forwarded-For
RemoteIPTrustedProxy 10.0.0.0/8
RemoteIPTrustedProxy 172.16.0.0/12
RemoteIPTrustedProxy 192.168.0.0/16
```

## Logging and Directory Options

Log format, the `/metrics` log exclusion and the `MultiViews` directory options are listed in `sites-deployment.md`. They live in `httpd-logs.conf` and `httpd-proxy.conf`; change them in both Dockerfiles.

## Adding a URL Block Rule

Append to the `RedirectMatch 403` block in `httpd-proxy.conf`. Use the allowlist exception pattern for files that must remain accessible:

```apache
RedirectMatch 403 ^(?!/(<allowed>)\.<ext>$).*\.<ext>/?$
```

## Adding a Convenience Redirect

Append to the redirect block in `httpd-proxy.conf`. Must be updated in both Dockerfiles (different target hosts).
