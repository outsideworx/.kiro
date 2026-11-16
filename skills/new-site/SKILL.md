---
name: new-site
description: Scaffold a new website — static Apache HTML/CSS/JS repo or dynamic npm app and register it in the platform. Use whenever the user wants to add, create, or set up a site, landing page, or frontend for a client; pick a splash/mobile/single-page entry pattern; write hotspot navigation or API fetch scripts; or wire a new site into sites/compose.yaml, Traefik, the build matrix, Prometheus, or the services_cache volume, including which network (edge vs outsideworx) it should join. Not for explaining how the existing network or stack works.
---

# New Site

## Overview

Each site is a standalone GitHub repo containing static HTML/CSS/JS. The `sites` repo's shared Dockerfile clones and serves it via Apache httpd. This skill covers creating the site repo itself — the frontend.

For backend wiring (API, database, compose, tokens), delegate to the `new-client` skill. For SEO files, delegate to the `seo` skill.

## Site Archetypes

Every site falls into one of these categories. Determine the archetype before scaffolding.

| Archetype | Example Sites | Has API | Key Feature |
|-----------|--------------|---------|-------------|
| Portfolio/Gallery | soupart, come-in-and-find-out | Yes | Paginated images from API, category navigation |
| Link Collector | duckumbrella, gaiapeeps | Optional | Social links or API-fetched embeds |
| Informational | igli, soupkitchen, outsideworx | No | Static content, no API calls of its own (outsideworx has an `APP_CLIENTS_WORX_TOKEN`, used to proxy `/api/` calls from the `/clients/<name>` mirror sites) |
| WIP (submodule) | thegreen | No (initially) | Hosted under outsideworx.net, client-secret protected |
| Dynamic npm app | tunde-divat | Self-contained | Node/Express + Vite SPA + own SQLite DB, built from `Dockerfile.npm`, port 80 |

The first four archetypes are **static** sites served by Apache from the shared `Dockerfile`. The **Dynamic npm app** archetype is fundamentally different: it is a full npm-workspaces monorepo whose Express server serves both its own REST API and the built SPA, with its own database — it does not use Apache, the shared static file layout, or the Spring Boot API. Most of this skill (entry-point patterns, hotspot navigation, `layout.css`, the API fetch/jQuery patterns) applies only to static sites. For an npm app, the repo's own `AGENTS.md` and `package.json` scripts define the structure; only `references/infra-wiring.md` applies.

## Required Files (All Sites)

```
<site-name>/
├── .github/workflows/build.yaml    # Dispatch to sites repo
├── favicon.ico                     # Site icon (.ico format)
├── img/                            # Image assets
│   └── flip.webp                   # "Rotate device" image (landscape sites only)
├── index.html                      # Entry point (splash or content)
├── metrics.txt                     # Health check ("up 1")
├── README.md                       # Site description + link + screenshot
├── robots.txt                      # SEO
├── sitemap.xml                     # SEO
└── style/
    └── layout.css                  # Core styles
```

## Entry Point Patterns

### Choosing a Pattern

| User says | Pattern | Reason |
|-----------|---------|--------|
| "No mobile view" / "landscape only" / "desktop only" | A | Content is designed for landscape screens; mobile users see a "rotate device" prompt |
| "Separate mobile design" / "different mobile layout" / "dedicated mobile view" | B | Desktop and mobile have fundamentally different visual designs (different images, different element arrangement) that can't be achieved with responsive CSS alone |
| "Mobile responsive" / "works on all devices" / "responsive" | C | Single HTML page adapts to all screen sizes via CSS media queries or flexible layout |

Rules:
- If the site uses full-bleed 1920×1080 background images with positioned hotspot navigation, it cannot be responsive — use Pattern A
- If the client provides separate desktop and mobile design mockups with different compositions, use Pattern B
- If the layout is text-based, grid-based, or uses standard web components (cards, lists, embeds), use Pattern C
- When in doubt and the user hasn't specified, ask

### Pattern A: Landscape-Only Splash (soupart, come-in-and-find-out, thegreen)

Used when the site is an image-driven experience designed exclusively for landscape viewports. The background images have fixed hotspot coordinates that don't translate to portrait/mobile. Mobile users are shown a "please rotate your device" image instead.

`index.html` is a minimal orientation gate — redirects to the content page in landscape mode, shows a "flip your device" image in portrait.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title><Site Name></title>
    <link rel="icon" type="image/x-icon" href="favicon.ico"/>
    <link rel="stylesheet" href="style/layout.css">
</head>
<body>
<div class="image-container">
    <img id="flipImgId" src="" width="400" alt="">
</div>
<script src="script/orientation/landscape.js"></script>
</body>
</html>
```

Requires:
- `img/flip.webp` — portrait-mode instruction image
- `script/orientation/landscape.js` — orientation check script
- A separate content page (e.g., `home.html` or `pages/home.html`)

### Pattern B: Mobile Redirect (duckumbrella, soupkitchen)

Used when the site has distinct desktop and mobile experiences — different image compositions, different element sizes or arrangements that go beyond what CSS media queries can handle. Both versions are full implementations, not just layout adjustments.

`index.html` serves desktop content directly but redirects mobile users to a separate `_mobile.html` variant.

```html
<script src="script/mobile.js"></script>
```

```javascript
// script/mobile.js
function isMobileDevice() {
    return /Mobi|Android/i.test(navigator.userAgent);
}

function updateLocation() {
    if (isMobileDevice()) {
        window.location.replace("index_mobile.html");
    }
}

updateLocation();
```

Requires:
- A `<page>_mobile.html` counterpart for each page that has a mobile variant
- The mobile script on every page that has a mobile counterpart
- Both versions share the same `style/` and `script/` directories where possible

### Pattern C: Single Page (gaiapeeps, igli, outsideworx)

Used when the content is text, embeds, cards, or other standard web elements that naturally reflow across screen sizes. No background-image navigation, no device-specific compositions.

`index.html` is the main content page — no splash, no redirect. Includes full meta tags directly. Responsive behaviour is handled via CSS media queries or flexible layout (flexbox/grid with `flex-wrap`).

## Where the Detail Lives

Read these on demand — they keep this file short:

| Need | File |
|------|------|
| Layout CSS, page template, hotspot `menu.js`, `landscape.js`, API fetch (jQuery), script layout, image assets | `references/static-patterns.md` (static sites only) |
| After any site add/remove: run `python3 scripts/check_sites.py <workspace-root>` — JSON report of mismatches between build matrix, both compose files and Prometheus targets | `scripts/check_sites.py` |
| compose (prod + test), `edge` vs `outsideworx`, npm-app compose variant, build matrix, Prometheus targets, cache volume | `references/infra-wiring.md` (authoritative; other skills delegate here) |

## GitHub Actions Workflow

Every site repo has exactly one workflow that dispatches a build event to the `sites` repo on push to `main`. See the `github` skill for the full setup steps and workflow template.

## README

```markdown
<Description paragraph — same text as the meta description, expanded to 3-5 sentences.>
[<domain>](https://<domain>)

---

![design_<name>](https://github.com/user-attachments/assets/<asset-id>)
```

The screenshot image is uploaded to GitHub as a release/issue asset and referenced by URL.

## CDN Dependencies

Sites use CDN-hosted libraries when needed:

| Library | Used When | CDN |
|---------|-----------|-----|
| jQuery | API fetch calls | `https://code.jquery.com/jquery-3.7.1.min.js` |
| Bootstrap CSS | Grid layouts, modals | `https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css` |
| Bootstrap JS | Modal interactions | `https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js` |

- Static-only sites with no API and no modals need no CDN dependencies
- jQuery is required for any site that calls `/api/`
- Bootstrap is optional — used for image modals and responsive grid layouts

## Prefetch / Preload

Content pages may include `<link rel="prefetch">` for likely next-navigation targets and `<link rel="preload">` for the page's own background image:

```html
<link rel="preload" as="image" href="img/<background>.webp">
<link rel="prefetch" href="<likely-next-page>.html">
```

## Checklist

When creating a new site repo:

1. Determine the archetype (portfolio, link collector, informational, WIP)
2. Create the repo structure with required files
3. Choose entry point pattern (A: landscape splash, B: mobile redirect, C: single page)
4. Create `favicon.ico` from client branding
5. Create background images at 1920×1080 (landscape sites) or appropriate size
6. Create `img/flip.webp` if using Pattern A
7. Write `layout.css` (core template in `references/static-patterns.md`)
8. Write navigation scripts with hotspot coordinates from the background image
9. Write API fetch scripts if the site has a backend (delegate wiring to `new-client` skill)
10. Create SEO files (`robots.txt`, `sitemap.xml`, `metrics.txt`)
11. Create `.github/workflows/build.yaml` with the dispatch event
12. Write README with description, link, and screenshot
13. Wire the site into the platform infrastructure (`references/infra-wiring.md`)
