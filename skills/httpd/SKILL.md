---
name: apache-httpd-sites
description: Apache httpd configuration conventions for the sites Dockerfile. Use when modifying the shared Dockerfile, adding security headers, changing proxy config, or adjusting rate limits.
---

# Apache httpd — Sites Configuration

## Overview

All seven sites share a single `Dockerfile` (prod) and `Dockerfile.test` (test). Apache config is generated inline via `RUN cat <<'EOF'` blocks — there are no external `.conf` files checked into the repo (except `blacklist.conf`).

## Image

- Base: `httpd:2.4`
- Two-stage build: `bitnami/git` fetches site content, `httpd:2.4` serves it

## Modules Enabled

Uncommented via `sed` in the Dockerfile:

- `headers` — request/response header manipulation
- `negotiation` — content negotiation (MultiViews)
- `proxy` + `proxy_http` — reverse proxy to services API
- `ratelimit` — output bandwidth throttling
- `remoteip` — trust X-Forwarded-For from Traefik
- `reqtimeout` — request read timeouts
- `unique_id` — generates unique request IDs

## Config File Structure

Includes appended to `httpd.conf` via `sed -e '$aInclude ...'`:

| File | Purpose |
|------|---------|
| `conf/extra/blacklist.conf` | IP deny rules (prod: real IPs, test: placeholder) |
| `conf/extra/httpd-logs.conf` | Log format and output |
| `conf/extra/httpd-token.conf` | TOKEN env var injection (prod only, created by entrypoint) |
| `conf/extra/httpd-proxy.conf` | Proxy, MPM, rate limit, headers, redirects, directory options |
| `conf/extra/httpd-remoteip.conf` | RemoteIP trusted proxy ranges |

## Entrypoint (prod only)

```sh
#!/bin/sh
if [ -n "${TOKEN}" ]; then
    echo "SetEnv TOKEN ${TOKEN}" > /usr/local/apache2/conf/extra/httpd-token.conf
else
    touch /usr/local/apache2/conf/extra/httpd-token.conf
fi
exec httpd-foreground
```

- Sites with API access have `TOKEN` set → written to `httpd-token.conf`
- Sites without API access → empty file (no error)
- Test Dockerfile skips this entirely (hardcodes `"test"` in proxy conf), uses `CMD ["httpd-foreground"]` directly

## Proxy Configuration

```apache
ProxyRequests Off
ProxyPreserveHost On
ProxyPass        "/api/"  "http://services_services/api/"      # prod
ProxyPass        "/api/"  "http://host.docker.internal:8080/api/"  # test
ProxyPassReverse "/api/"  "<same as ProxyPass>"
```

## Request Headers

```apache
RequestHeader set X-Auth-Token "${TOKEN}"    # prod (from env)
RequestHeader set X-Auth-Token "test"        # test (hardcoded)
RequestHeader set X-Caller-Id "${NAME}"
RequestHeader set X-Request-Id "%{UNIQUE_ID}e"
```

## Response Headers

```apache
Header always set X-Request-Id "%{UNIQUE_ID}e"
Header always set X-Content-Type-Options "nosniff"
Header always set X-Frame-Options "DENY"
Header always set Referrer-Policy "strict-origin-when-cross-origin"
Header always set Content-Security-Policy "..."
```

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

## MPM Event Tuning

| Setting | Prod | Test |
|---------|------|------|
| StartServers | 2 | 1 |
| MinSpareThreads | 16 | 4 |
| ThreadsPerChild | 64 | 4 |
| MaxRequestWorkers | 128 | 4 |

## Rate Limiting

```apache
SetOutputFilter RATE_LIMIT
SetEnv rate-limit 1536    # prod (KB/s)
SetEnv rate-limit 640     # test (KB/s)
```

## Request Timeouts

```apache
RequestReadTimeout header=2-5,MinRate=2048 body=5-30,MinRate=4096
```

Same in both prod and test.

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

## URL Blocking

```apache
RedirectMatch 403 /\.
RedirectMatch 403 \.(bak|conf|config|env|ini|json|key|log|properties|php|pub|py|sh|ts|yaml|yml|zip)/?$
RedirectMatch 403 ^(?!/(metrics|robots)\.txt$).*\.txt/?$
RedirectMatch 403 ^(?!/(sitemap)\.xml$).*\.xml/?$
```

## Convenience Redirects

```apache
RedirectMatch 301 ^/grafana/?$  https://services.outsideworx.net/grafana   # prod
RedirectMatch 301 ^/login/?$    https://services.outsideworx.net           # prod
RedirectMatch 301 ^/ntfy/?$     https://services.outsideworx.net/ntfy      # prod
```

Test uses `http://localhost:8080/...` instead.

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

## Prod vs Test Differences Summary

| Aspect | Prod (`Dockerfile`) | Test (`Dockerfile.test`) |
|--------|---------------------|--------------------------|
| Blacklist | `blacklist.conf` | `blacklist-test.conf` |
| Token injection | Entrypoint writes `httpd-token.conf` | Hardcoded `"test"` in proxy conf |
| Includes `httpd-token.conf` | Yes | No |
| Proxy target | `services_services` (Swarm DNS) | `host.docker.internal:8080` |
| Entrypoint | Custom shell script | `httpd-foreground` directly |
| Rate limit | 1536 KB/s | 640 KB/s |
| MPM workers | 128 max | 4 max |
| Redirects | `https://services.outsideworx.net/...` | `http://localhost:8080/...` |
