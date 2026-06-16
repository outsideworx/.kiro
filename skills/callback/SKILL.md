---
name: callback
description: Contact form callback system (visitor submits contact info → email sent → persisted to DB). Use when adding callback functionality to a new client or modifying the existing callback flow.
---

# Callback System

## Overview

The callback system lets site visitors express interest in a product by submitting their contact info. The backend sends a notification email to the client and persists the callback to the database for record-keeping.

## Flow

```
Visitor clicks "Callback" on site
        │
        ▼
Frontend modal (email input + submit button)
        │
        ▼ POST /api/callback (JSON body)
Apache ProxyPass → services (X-Auth-Token, X-Caller-Id, X-Request-Id injected)
        │
        ▼
CallbackController
        ├── EmailGateway.send() → MailerSend API → email to <callerId>@outsideworx.net
        └── CallbackRepository.save() → CALLBACK table (always, even if email fails)
```

## Components

| Component | Location | Purpose |
|-----------|----------|---------|
| `Callback` | `models/Callback.java` | DTO — request body (`address`, `product`) |
| `CallbackEntity` | `models/CallbackEntity.java` | JPA entity — persisted record |
| `CallbackRepository` | `repositories/CallbackRepository.java` | Plain `CrudRepository` |
| `CallbackController` | `controllers/CallbackController.java` | Receives POST, orchestrates email + persistence |
| `EmailGateway` | `gateways/EmailGateway.java` | Sends email via MailerSend SDK |

## Request

```
POST /api/callback
Content-Type: application/json
X-Auth-Token: <injected by Apache>
X-Caller-Id: <injected by Apache>
```

```json
{
  "address": "visitor@example.com",
  "product": "https://come-in-and-find-out.ch/pages/details?id=42"
}
```

## Controller Logic

```java
@CrossOrigin({"${app.clients.ciafo.origin}"})
@RestController
@RequiredArgsConstructor
@Slf4j
final class CallbackController {
    private final CallbackRepository callbackRepository;

    private final EmailGateway emailGateway;

    @PostMapping("/api/callback")
    void callback(@RequestHeader("X-Caller-Id") String callerId, @RequestBody Callback callback) {
        // 1. Send email (may throw)
        // 2. Always persist to DB (in finally block)
    }
}
```

Key behaviors:
- Email is sent first; if it fails, an `IllegalStateException` is thrown
- The DB save happens in a `finally` block — the callback is persisted regardless of email success/failure
- CORS is restricted to the client's origin (`app.clients.ciafo.origin`)

## Email

`EmailGateway.send(callerId, subject, text)`:
- From: `Outside Worx <info@outsideworx.net>`
- To: `<callerId>@outsideworx.net` (e.g., `come-in-and-find-out@outsideworx.net`)
- Subject: `"Someone is interested!"`
- Body: HTML with the visitor's contact address and a link to the product

Requires `MAILERSEND_SDK_TOKEN` environment variable (injected via compose → Spring property `mailersend.sdk.token`).

## Database

Table: `CALLBACK`

| Column | Type | Description |
|--------|------|-------------|
| `id` | BIGINT (PK) | Auto-generated |
| `address` | VARCHAR(255) | Visitor's contact (email/phone) |
| `processed` | BOOLEAN | Whether the client has followed up (nullable, set manually) |
| `product` | VARCHAR(255) | URL of the product the visitor was interested in |
| `recipient` | VARCHAR(255) | The `X-Caller-Id` value (site name) |

No hash trigger — this table has no image columns.

## Frontend Integration

The site provides a modal with an input field and submit button:

```javascript
$.ajax({
    url: "/api/callback",
    method: "POST",
    contentType: "application/json",
    data: JSON.stringify({ address: address, product: window.location.href }),
    success: function () { /* show success message */ },
    error: function () { /* show fallback contact info */ }
});
```

The `product` field is always `window.location.href` — the current page URL.

## Adding Callback to a New Client

1. Add the client's origin to `@CrossOrigin` on `CallbackController` (comma-separated list):
   ```java
   @CrossOrigin({"${app.clients.ciafo.origin}", "${app.clients.<NEW>.origin}"})
   ```

2. Add the frontend modal and JS to the site (copy the pattern from come-in-and-find-out's `script/fetch/callback.js`)

3. Ensure the site has a `TOKEN` environment variable (required for `X-Auth-Token` — already configured if the site calls any API)

No changes needed to the entity, repository, gateway, or database schema — the system is already multi-tenant via the `recipient` column.
