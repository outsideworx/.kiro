---
name: new-site
description: Scaffolding a new site repo. Use when creating the frontend repo for a new or existing client — covers file structure, page types, assets, scripts, and CI.
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
| Informational | igli, soupkitchen, outsideworx | No | Static content, no API calls |
| WIP (submodule) | thegreen | No (initially) | Hosted under outsideworx.net, client-secret protected |

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

## Page Structure (Landscape/Image-Based Sites)

### Layout CSS (Core)

All landscape-based sites share this base `layout.css`:

```css
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
    overflow: hidden;
    -webkit-touch-callout: none;
    -webkit-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    user-select: none;
}

.image-container {
    position: relative;
    width: 100vw;
    height: 100vh;
    display: flex;
    justify-content: center;
    align-items: center;
    overflow: hidden;
}

@media (min-aspect-ratio: 1.9) {
    .image-container {
        height: 100dvh;
    }
}

.image-container img {
    max-width: 100%;
    max-height: 100%;
    object-fit: contain;
}

.link {
    position: absolute;
    background: rgba(255, 255, 255, 0.0);
    cursor: pointer;
}
```

### Content Page Template

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="description" content="<description>"/>
    <meta name="author" content="Outside Worx"/>
    <meta name="robots" content="index, follow">
    <title><Site Name></title>
    <link rel="icon" type="image/x-icon" href="favicon.ico"/>
    <link rel="stylesheet" href="style/layout.css">
</head>
<body>
<div class="image-container">
    <img id="main-image" src="img/<background>.webp" alt="">
    <!-- Navigation links positioned over the background -->
    <a href="<target>.html" class="link" id="<name>"></a>
</div>
<script src="script/navigation/menu.js"></script>
</body>
</html>
```

### Background Color

Set `<body style="background-color: <color>">` to match the dominant color of the background image. This prevents white flash during load and fills letterboxing on ultra-wide screens.

Alternatively, use `body` background in a separate CSS file (e.g., `colors-<page>.css`):

```css
html, body {
    height: 100%;
    width: 100%;
    background-color: <color>;
    display: flex;
    justify-content: center;
    align-items: center;
}
```

## Navigation (Hotspot Positioning)

**Only applies to Pattern A (landscape/image-based) sites.**

Sites built entirely around full-bleed background images use invisible positioned `<a>` elements overlaid on the image as navigation. Standard sites (Pattern B and C) use regular HTML links, buttons, or nav elements — no hotspot positioning needed.

### menu.js Template

```javascript
function positionLinks() {
    const img = document.getElementById("main-image");

    const imgNaturalWidth = 1920;
    const imgNaturalHeight = 1080;
    const navigation = [
        {element: document.getElementById("<id>"), x: <X>, y: <Y>, width: <W>}
    ];

    const imgContainer = img.parentElement;
    const containerRect = imgContainer.getBoundingClientRect();

    const scale = Math.min(containerRect.width / imgNaturalWidth, containerRect.height / imgNaturalHeight);

    const displayedWidth = imgNaturalWidth * scale;
    const displayedHeight = imgNaturalHeight * scale;

    const offsetX = (containerRect.width - displayedWidth) / 2;
    const offsetY = (containerRect.height - displayedHeight) / 2;

    navigation.forEach(hs => {
        hs.element.style.left = `${offsetX + hs.x * scale}px`;
        hs.element.style.top = `${offsetY + hs.y * scale}px`;
        hs.element.style.width = `${(hs.width || 200) * scale}px`;
        hs.element.style.height = `${(hs.height || 200) * scale}px`;
    });
}

window.addEventListener("resize", positionLinks);
window.addEventListener("load", positionLinks);
```

- Coordinates are absolute pixel positions on the 1920×1080 source image
- Multiple navigation scripts can coexist (e.g., `menu.js` for main nav, `navigation.js` for back/forward)
- Each page gets its own navigation config (different hotspot positions)

## Orientation Script

### landscape.js Template

```javascript
function checkOrientation() {
    if (window.innerWidth > window.innerHeight) {
        window.location.href = "<content-page>";
    } else {
        document.getElementById("flipImgId").src = "img/flip.webp"
    }
}

window.addEventListener("load", checkOrientation);
window.addEventListener("resize", checkOrientation);
```

The redirect target is the main content page (e.g., `home.html`, `pages/home.html`).

## API Fetch Pattern (Sites with Backend)

Sites that call the API use jQuery `$.ajax`:

### Paginated Gallery Fetch

```javascript
function loadImages(category) {
    const urlParams = new URLSearchParams(window.location.search);
    const offset = Number(urlParams.get("offset") || 0);
    $.ajax({
        url: `/api/cache/<site-name>?category=${category}&offset=${offset}`,
        method: 'GET',
        success: function (response) {
            if (response && Array.isArray(response)) {
                if (response.length === 0 && offset !== 0) {
                    window.history.go(-1);
                }
                response.forEach((item, index) => {
                    // Populate DOM elements with response data
                });
                // Remove unused placeholder elements
            }
        },
        error: function (error) {
            console.error('Error fetching images:', error);
        }
    });
}
```

### Simple List Fetch (No Pagination)

```javascript
function loadItems() {
    $.ajax({
        url: "/api/<site-name>",
        method: "GET",
        success: function(items) {
            const container = $("#<container-id>");
            container.empty();
            items.forEach(item => {
                // Build and append DOM elements
            });
        },
        error: function(xhr, status, error) {
            console.error("Error loading items:", status, error);
        }
    });
}
$(document).ready(loadItems);
```

### Conventions

- jQuery (`$.ajax`) for all API calls — loaded from CDN
- Relative URLs (`/api/...`) — Apache proxy handles routing
- Empty response with non-zero offset → `window.history.go(-1)` (navigate back)
- Remove unused placeholder elements from DOM (don't just hide them)
- `console.error` on failure (no user-facing error UI unless the site has a callback modal)

## Script Directory Structure

```
script/
├── fetch/              # API calls (only for sites with backend)
│   └── images.js       # Or categories.js, details.js, callback.js
├── navigation/         # Hotspot positioning
│   ├── menu.js         # Main page navigation
│   └── navigation.js   # Sub-page back/forward
├── orientation/        # Device/viewport handling
│   ├── landscape.js    # Landscape gate
│   └── mobile.js       # Mobile size adjustments
└── effect/             # Optional visual effects
    └── <effect>.js     # Site-specific (e.g., lightning.js, mousedown.js)
```

Not all directories are needed — only include what the site uses.

## Image Assets

### Format

- Background images: `.webp`, 1920×1080 pixels (landscape)
- Flip image: `img/flip.webp` (portrait-mode instruction, shared design)
- Logos/icons: `.webp` or `.png`
- Favicon: `.ico` format at repo root

### Naming

- Backgrounds: descriptive name (`background_0.webp`, `art.webp`, `home.webp`)
- Numbered variants: `_0`, `_1` suffix for states (default vs. active)
- Per-page backgrounds: named after the page they serve

### Size Budget

Background images are typically 75KB–425KB each (webp compression). Favicons range 15KB–190KB.

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
7. Write `layout.css` (use core template for landscape sites)
8. Write navigation scripts with hotspot coordinates from the background image
9. Write API fetch scripts if the site has a backend (delegate wiring to `new-client` skill)
10. Create SEO files (`robots.txt`, `sitemap.xml`, `metrics.txt`)
11. Create `.github/workflows/build.yaml` with the dispatch event
12. Write README with description, link, and screenshot
13. Wire the site into `sites/compose.yaml`, `sites/compose-test.yaml`, build matrix, and Prometheus
