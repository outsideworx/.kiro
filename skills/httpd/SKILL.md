---
name: httpd
description: Apache httpd configuration conventions for the sites Dockerfile. Use when modifying the shared Dockerfile, adding security headers, changing proxy config, or adjusting rate limits.
---

# Apache httpd — Sites Configuration

## Overview

All sites share a single `Dockerfile` (prod) and `Dockerfile.test` (test). Apache config is generated inline via `RUN cat <<'EOF'` blocks — there are no external `.conf` files checked into the repo (except `blacklist.conf`).

For the current configuration values (modules, headers, rate limits, timeouts, URL blocking, redirects, prod vs test differences), see `sites-deployment.md`.

## Config File Structure

Includes appended to `httpd.conf` via `sed -e '$aInclude ...'`:

| File | Purpose |
|------|---------|
| `conf/extra/blacklist.conf` | IP deny rules (prod: real IPs, test: placeholder) |
| `conf/extra/httpd-auth.conf` | Client secret Lua auth (prod only, created by entrypoint) |
| `conf/extra/httpd-logs.conf` | Log format and output |
| `conf/extra/httpd-token.conf` | TOKEN env var injection (prod only, created by entrypoint) |
| `conf/extra/httpd-proxy.conf` | Proxy, MPM, rate limit, headers, redirects, directory options |
| `conf/extra/httpd-remoteip.conf` | RemoteIP trusted proxy ranges |

## Entrypoint (prod only)

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
- Test Dockerfile has its own simpler entrypoint (no `httpd-token.conf`, hardcodes `"test"` in proxy conf)

## Enabling a Module

Modules are uncommented via `sed` in the Dockerfile:

```dockerfile
RUN sed -i \
    -e 's/^#LoadModule headers_module/LoadModule headers_module/' \
    -e 's/^#LoadModule lua_module/LoadModule lua_module/' \
    conf/httpd.conf
```

To add a new module, append another `-e` line to this `sed` block (alphabetical order).

## Proxy Configuration

```apache
ProxyRequests Off
ProxyPreserveHost On
ProxyPass        "/api/"  "http://services_services/api/"      # prod
ProxyPass        "/api/"  "http://host.docker.internal:8080/api/"  # test
ProxyPassReverse "/api/"  "<same as ProxyPass>"
```

## Response Headers

```apache
Header always set X-Request-Id "%{UNIQUE_ID}e"
Header always set X-Content-Type-Options "nosniff"
Header always set X-Frame-Options "DENY"
Header always set Referrer-Policy "strict-origin-when-cross-origin"
Header always set Content-Security-Policy "..."
```

To add a new response header, append to this block in `httpd-proxy.conf` (alphabetical by header name after `X-Request-Id`).

### Content-Security-Policy

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

## Logging

All default Apache log directives are stripped via `find`/`sed`, then replaced:

```apache
ErrorLogFormat "ERROR %P --- ip=%a requestId=%{UNIQUE_ID}e: %M"
ErrorLog /proc/self/fd/2
LogFormat "INFO %P --- ip=%a requestId=%{UNIQUE_ID}e: %r %>s" log_format
SetEnvIf Request_URI "^/metrics$" no_log
CustomLog /proc/self/fd/1 log_format env=!no_log
```

- Logs go to stdout/stderr (Docker captures them)
- `/metrics` requests excluded from access log

## Directory Options

```apache
<Directory "/usr/local/apache2/htdocs">
    Options +MultiViews
</Directory>
<Directory "/usr/local/apache2/htdocs/clients">
    Options +MultiViews
    DirectoryIndex index.html
</Directory>
```

MultiViews enables extensionless URL serving (content negotiation).

## Adding a URL Block Rule

Append to the `RedirectMatch 403` block in `httpd-proxy.conf`. Use the allowlist exception pattern for files that must remain accessible:

```apache
RedirectMatch 403 ^(?!/(<allowed>)\.<ext>$).*\.<ext>/?$
```

## Adding a Convenience Redirect

Append to the redirect block in `httpd-proxy.conf`. Must be updated in both Dockerfiles (different target hosts).
