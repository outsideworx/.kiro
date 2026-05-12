# New User Registration

## Overview

Adding a new user touches multiple systems. There is no unified user management — each service maintains its own user store.

## Systems Involved

| System | User store | Format |
|--------|-----------|--------|
| Authelia | `authelia-users.yaml` / `authelia-test-users.yaml` | YAML (argon2id hashed passwords) |
| ntfy | SQLite database (`/etc/ntfy/data/user.db`) | CLI-managed (`ntfy user add`) |
| Services (Spring Boot) | No user table — routes by email domain | Thymeleaf template must exist |

## Step-by-Step

### 1. Authelia — Add User

Add entry to `authelia-users.yaml` (prod) and `authelia-test-users.yaml` (test):

```yaml
users:
  info@<domain>:
    displayname: <DISPLAY_NAME>
    email: info@<domain>
    password: "<argon2id hash>"
```

- The email address determines which portal view the user sees (domain extraction: `(?<=@)[^.]+(?=\\.)`)
- Users without the `admin` group get `Viewer` role in Grafana
- Generate password hash: `docker run --rm authelia/authelia:4.37 authelia crypto hash generate argon2 --password '<password>'`
- In test config, all client passwords can share the same hash for convenience

### 2. ntfy — Add User

Prod (on the server):
```bash
docker exec -it <ntfy-container> ntfy user add --role=user <username>
```

This modifies `/etc/ntfy/data/user.db` directly. The utils sidecar also mounts this volume.

Test: ntfy uses hardcoded users/tokens in `ntfy-test.yaml` — no SQLite management needed:
```yaml
auth-tokens:
  - "admin:tk_...:proxy"
auth-users:
  - "admin:$2b$...:admin"
```

In test mode, ntfy auth is simplified — tokens and users are defined inline in the config file rather than in the database.

### 3. Services — Create Landing Page

The `IndexController` extracts the domain from the user's email and looks for a `ModelVisitor` whose view name matches `clients/<domain>`. If no match exists, it throws `AccessDeniedException` and the `ErrorController` redirects to Authelia.

To add a new user's portal view:

1. Create a Thymeleaf template: `src/main/resources/templates/clients/<domain>.html`
2. Create a controller implementing `ModelVisitor` that returns `ModelAndView("clients/<domain>")`
3. Without this, the user can log in via Authelia but will hit the error controller on every request

### 4. Optional — API Client Setup

If the new user's site needs API access (not all do):

1. Add client config to `application.yaml` and `application-test.yaml` under `app.clients`
2. Add `APP_CLIENTS_<NAME>_TOKEN` to services `.env` and sites `.env`
3. Add `TOKEN` environment variable to the site's compose entry
4. Create API controller, repository, entity, converter (see spring-boot.md client pattern)

## Prod vs Test

| System | Prod | Test |
|--------|------|------|
| Authelia | `authelia-users.yaml`, real argon2id hashes | `authelia-test-users.yaml`, shared test hashes |
| ntfy | SQLite DB managed via CLI | Inline in `ntfy-test.yaml` (no DB interaction) |
| Services portal | Must have matching template | Same requirement |
| API tokens | Unique per client from `.env` | All set to `"test"` |

## Common Pitfall

If a user can authenticate via Authelia but sees a redirect loop or error page in the services portal, the most likely cause is a missing Thymeleaf template for their email domain. The `ErrorController` logs `"Supressing Whitelabel Error Page"` and redirects to Authelia, creating an apparent loop.
