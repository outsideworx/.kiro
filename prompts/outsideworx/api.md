# API Reference

## Overview

The Spring Boot backend exposes REST endpoints under `/api/`. Static sites proxy requests through Apache (`ProxyPass "/api/"`) with `X-Auth-Token` and `X-Caller-Id` headers injected automatically. The frontend JavaScript calls these endpoints as relative paths (`/api/...`) — the proxy is transparent.

## `/api/` vs `/api/cache/`

| Prefix | Cache-Control Header | Use Case |
|--------|---------------------|----------|
| `/api/` | None | Data that changes frequently or needs fresh responses |
| `/api/cache/` | `public, max-age=86400` (24h) | Data that rarely changes — detail views, paginated galleries |

The `CacheFilter` applies the 24h header to any request matching `/api/cache/**`. Both prefixes go through the same `AuthTokenFilter` validation.

## Endpoints

### come-in-and-find-out

#### `GET /api/come-in-and-find-out`

Paginated preview list for a category.

| Param | Type | Description |
|-------|------|-------------|
| `category` | String | Category name (e.g., `"Accessories"`, `"Art"`) |
| `offset` | int | Zero-based offset for pagination (page size: 6) |

Response: `List<CiafoPreview>`

```json
[
  {
    "id": 42,
    "category": "Art",
    "description": "...",
    "hash": "a1b2c3d4",
    "image1": "data:image/jpeg;base64,..."
  }
]
```

Returns empty array when offset exceeds available items (frontend navigates back).

#### `GET /api/cache/come-in-and-find-out`

Full detail payload for a single item.

| Param | Type | Description |
|-------|------|-------------|
| `id` | Long | Item ID |

Response: `CiafoPayload`

```json
{
  "id": 42,
  "category": "Art",
  "description": "...",
  "hash": "a1b2c3d4",
  "image1": "data:image/jpeg;base64,...",
  "image2": "data:image/jpeg;base64,...",
  "image3": null,
  "image4": null
}
```

Null fields are omitted from the response (`NON_EMPTY` Jackson setting).

#### `POST /api/callback`

Contact callback — visitor leaves contact info for a product they're interested in.

Request body:

```json
{
  "address": "user@example.com",
  "product": "https://come-in-and-find-out.ch/pages/details?id=42"
}
```

Response: `void` (200 OK on success)

Side effects:
- Sends email to `<callerId>@outsideworx.net` via MailerSend
- Persists callback to the `CALLBACK` table

Only come-in-and-find-out uses this endpoint (CORS restricted to its origin).

### soupart

#### `GET /api/cache/soupart`

Paginated item list for a category.

| Param | Type | Description |
|-------|------|-------------|
| `category` | String | Category name (e.g., `"art"`, `"illustration"`, `"animation"`, `"design"`) |
| `offset` | int | Zero-based offset for pagination (page size: 9) |

Response: `List<SoupEntity>`

```json
[
  {
    "id": 15,
    "category": "art",
    "description": "...",
    "hash": "e5f6g7h8",
    "image": "data:image/jpeg;base64,...",
    "thumbnail": "data:image/jpeg;base64,...",
    "link": "https://..."
  }
]
```

The frontend uses `thumbnail` for grid display and `image` or `link` for the full view (animation category uses `link` for video URLs).

### gaiapeeps

#### `GET /api/gaiapeeps`

All items (no pagination, no categories).

Response: `List<PeepsEntity>`

```json
[
  {
    "id": 1,
    "link": "https://youtube.com/...",
    "title": "Video Title"
  }
]
```

## Response Shape Reference

### Interface Projections (ciafo)

Ciafo uses interface-based projections to select only needed columns:

| Interface | Extends | Fields |
|-----------|---------|--------|
| `CiafoMeta` | — | `id`, `category`, `description`, `hash` |
| `CiafoPreview` | `CiafoMeta` | + `image1` |
| `CiafoPayload` | `CiafoPreview` | + `image2`, `image3`, `image4` |
| `CiafoThumbnails` | `CiafoMeta` | + `thumbnail1`, `thumbnail2`, `thumbnail3`, `thumbnail4` |

### Full Entities (soup, peeps)

Soupart and gaiapeeps return the full entity. Null/empty fields are omitted from JSON via `spring.jackson.default-property-inclusion: NON_EMPTY`.

## Pagination

| Client | Page Size | Mechanism |
|--------|-----------|-----------|
| come-in-and-find-out | 6 | `LIMIT 6 OFFSET :offset` |
| soupart | 9 | `LIMIT 9 OFFSET :offset` |
| gaiapeeps | — | No pagination (returns all) |

Frontend passes `offset` as a URL query parameter. Navigation links increment/decrement by page size. When the response is an empty array and offset > 0, the frontend navigates back.

## Which Site Calls What

| Site | Endpoints Used | Has Pagination |
|------|---------------|----------------|
| come-in-and-find-out | `/api/come-in-and-find-out`, `/api/cache/come-in-and-find-out`, `/api/callback` | Yes (6) |
| soupart | `/api/cache/soupart` | Yes (9) |
| gaiapeeps | `/api/gaiapeeps` | No |
| duckumbrella | — | — |
| igli | — | — |
| outsideworx | — | — |
| soupkitchen | — | — |

## Metrics

Every API request registers a Micrometer counter via `GrafanaGateway.registerRequest(endpoint, fetch)`:

| Endpoint | `endpoint` tag | `fetch` tag |
|----------|---------------|-------------|
| `GET /api/come-in-and-find-out` | `come-in-and-find-out` | category value |
| `GET /api/cache/come-in-and-find-out` | `come-in-and-find-out` | `details` |
| `GET /api/cache/soupart` | `soupart` | category value |
| `GET /api/gaiapeeps` | `gaiapeeps` | `all` |

Counter name: `services_requests` with tags `endpoint` and `fetch`.
