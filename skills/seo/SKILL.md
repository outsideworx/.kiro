---
name: seo
description: SEO conventions (robots.txt, sitemap.xml, metrics.txt, meta tags). Use when creating a new site repo or modifying SEO-related files.
---

# SEO

## Overview

Every site repo contains three SEO/infrastructure files at the root: `robots.txt`, `sitemap.xml`, and `metrics.txt`. Content pages include HTML meta tags for search engines. Splash pages (orientation gates) have no meta tags.

## Required Files

### robots.txt

Restrictive by default — blocks crawling of everything except the main content page, sitemap, and favicon.

#### Template

```
User-agent: *
Disallow: /
Allow: /sitemap.xml
Allow: /<main-page>
Allow: /favicon.ico

Sitemap: https://<domain>/sitemap.xml
```

#### Rules

- `Disallow: /` blocks all paths by default
- `Allow` entries whitelist only the pages meant for indexing
- The main page path matches what's in `sitemap.xml`
- `/favicon.ico` is allowed (omitted on some older sites but should always be included)
- `Sitemap:` line always points to the full HTTPS URL
- WIP sites under a subpath use the parent domain: `Sitemap: https://outsideworx.net/clients/<name>/sitemap.xml`

#### Main Page Patterns

| Site Type | Main Page | Example |
|-----------|-----------|---------|
| Has splash (`index.html`) + content page | `/home.html` | soupart, duckumbrella |
| Has splash + nested content | `/pages/home/<page>.html` | come-in-and-find-out |
| Single-page (no splash) | `/index.html` | gaiapeeps, igli, soupkitchen, outsideworx |
| WIP submodule | `/pages/home.html` | thegreen |

### sitemap.xml

Single-URL sitemap pointing to the main crawlable page.

#### Template

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
    <url>
        <loc>https://<domain>/<path></loc>
        <lastmod>YYYY-01-01</lastmod>
        <changefreq>monthly</changefreq>
        <priority>1.0</priority>
    </url>
</urlset>
```

#### Rules

- One `<url>` entry only — the main content page
- `<loc>` uses extensionless URLs (Apache MultiViews resolves them): `/home` not `/home.html`
- Exception: if the main page is `index.html` at the root, `<loc>` is just the domain with no path
- `<lastmod>` uses `YYYY-01-01` format (updated to current year at launch)
- `<changefreq>` is always `monthly`
- `<priority>` is always `1.0`
- WIP sites use the full subpath: `https://outsideworx.net/clients/<name>/pages/home`

#### Loc Patterns

| Site | `<loc>` |
|------|---------|
| soupart | `https://soupart.net/home` |
| come-in-and-find-out | `https://come-in-and-find-out.ch/pages/home/red_home` |
| gaiapeeps | `https://gaiapeeps.com` |
| duckumbrella | `https://duckumbrella.net/home` |
| igli | `https://igli.info` |
| soupkitchen | `https://soupkitchen.info` |
| outsideworx | `https://outsideworx.net` |
| thegreen (WIP) | `https://outsideworx.net/clients/thegreen/pages/home` |

### metrics.txt

Static health check file used by Prometheus for site-level scraping.

```
up 1
```

- Exact content: `up 1` (no trailing newline beyond the standard file-ending newline)
- Located at the site root
- Excluded from Apache access log via `SetEnvIf Request_URI "^/metrics$" no_log`
- Blocked from public access by URL blocking rules — except the explicit allowance: `RedirectMatch 403 ^(?!/(metrics|robots)\.txt$).*\.txt/?$`

## HTML Meta Tags

### Content Pages (crawlable)

Pages listed in `robots.txt` Allow rules include the full meta tag set:

```html
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="description" content="<site-specific description>"/>
    <meta name="author" content="Outside Worx"/>
    <meta name="robots" content="index, follow">
    <title><Site Name></title>
    <link rel="icon" type="image/x-icon" href="favicon.ico"/>
    ...
</head>
```

#### Tag Order

1. `charset`
2. `viewport`
3. `description`
4. `author`
5. `robots`
6. `<title>`
7. `<link rel="icon">`
8. Stylesheets and scripts

#### Rules

- `description` — unique per site, describes the site's purpose in one sentence
- `author` — always `Outside Worx`
- `robots` — always `index, follow` on content pages
- `viewport` — always `width=device-width, initial-scale=1.0`
- `favicon` path is relative; use `../favicon.ico` for nested pages

### Splash Pages (not crawlable)

Orientation-gate pages (`index.html` that redirect based on device) have minimal heads:

```html
<head>
    <meta charset="UTF-8">
    <title><Site Name></title>
    <link rel="icon" type="image/x-icon" href="favicon.ico"/>
    <link rel="stylesheet" href="style/layout.css">
</head>
```

- No `viewport`, `description`, `author`, or `robots` meta tags
- These pages are blocked by `robots.txt` (`Disallow: /` covers them)

### Internal Pages (not crawlable)

Category pages, detail pages, and other navigation pages include the full meta set but are not explicitly allowed in `robots.txt`. They use `"index, follow"` anyway (in case linked from an allowed page) and include a proper `description`.

## WIP Site Differences

WIP sites hosted as submodules under `outsideworx.net/clients/<name>/`:

- `robots.txt` Sitemap URL uses the parent domain + subpath
- `sitemap.xml` loc uses the parent domain + subpath
- Relative paths in HTML (`../favicon.ico`, `../style/`, `../script/`) since pages are nested
- No separate compose entry — shares the `outsideworx` container

## Checklist

When creating SEO files for a new site:

1. Determine the main content page path (splash → content, or single-page)
2. Create `robots.txt` with Allow for the main page, sitemap, and favicon
3. Create `sitemap.xml` with extensionless loc URL pointing to the main page
4. Create `metrics.txt` containing `up 1`
5. Add full meta tags to the main content page (`description`, `author`, `robots`, `viewport`)
6. Add minimal meta tags to splash/orientation pages (charset + title only)
7. Verify the domain matches what will be configured in Traefik labels
