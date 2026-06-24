---
name: new-client
description: Adding a new client to the platform. Use when onboarding a new client that needs API access and database storage. May include an admin portal view.
---

# New Client Registration

## Overview

Adding a new client touches multiple systems. This skill covers the full wiring — from tokens to API to compose config. For database schema creation, delegate to the `liquibase` skill. For admin portal templates, delegate to the `admin-portal` skill.

Steps 1 (Authelia) and 5's admin controller are only needed if the client will have an admin portal view. Clients without a portal still need the entity, repository, converter, and API controller.

## Systems Involved

| System | What to create |
|--------|----------------|
| Authelia | User entry in `authelia-users.yaml` + `authelia-test-users.yaml` |
| application.yaml | Client entry under `app.clients` |
| compose.yaml (services) | `APP_CLIENTS_<NAME>_TOKEN` environment variable |
| compose.yaml (sites) | `TOKEN` environment variable on the site service |
| .env (services + sites) | Token values |
| ntfy | User account (prod only, via CLI) |
| Spring Boot | Entity, Repository, Converter, ApiController, Controller (ModelVisitor) |
| Thymeleaf | Admin template (delegate to `admin-portal` skill) |
| Liquibase | Changelog (delegate to `liquibase` skill) |

## Step-by-Step

### 1. Authelia Users

Add entry to both `authelia-users.yaml` (prod) and `authelia-test-users.yaml` (test):

```yaml
  info@<domain>:
    displayname: <DISPLAY_NAME>
    email: info@<domain>
    password: "<argon2id hash>"
```

- The email domain determines which portal view the user sees (regex: `(?<=@)[^.]+(?=\\.)`)
- Generate prod hash: `docker run --rm authelia/authelia authelia crypto hash generate argon2 --password '<password>'`
- Test users all share the same hash (password: `test`): `$argon2id$v=19$m=65536,t=3,p=4$vSWXSYTyISnsOUqnNw6PQA$FvOvwV2ASM4Ii5ax2pt+lMAY+01DpZEtX+50IjPFhBw`

### 2. Application Config

Add client to `app.clients` in `application.yaml`:

```yaml
app:
  clients:
    <CLIENT>:
      caller: "<site-name>"
      origin: "https://<domain>"
      token: ${APP_CLIENTS_<CLIENT>_TOKEN}
```

Add test override in `application-test.yaml`:

```yaml
app:
  clients:
    <CLIENT>:
      origin: "*"
      token: "test"
```

### 3. Environment Variables

Services `compose.yaml` — add to the `services` service environment:

```yaml
APP_CLIENTS_<CLIENT>_TOKEN: $APP_CLIENTS_<CLIENT>_TOKEN
```

Services `.env` — add the token value:

```
APP_CLIENTS_<CLIENT>_TOKEN=<generated-token>
```

Sites `compose.yaml` — add `TOKEN` to the site service:

```yaml
environment:
  TOKEN: $APP_CLIENTS_<CLIENT>_TOKEN
```

Sites `.env` — add the same token:

```
APP_CLIENTS_<CLIENT>_TOKEN=<same-token>
```

### 4. ntfy User (prod only)

```bash
docker exec -it <ntfy-container> ntfy user add --role=user <username>
```

Not needed in test — test config uses a single admin token with login disabled.

### 5. Spring Boot Code

Create the following in `src/main/java/net/outsideworx/services/`:

#### Entity (`models/clients/<name>/<Name>Entity.java`)

```java
@Data
@Entity
@Table(name = "<UPPER_TABLE>")
public final class <Name>Entity {
    @Id
    @GeneratedValue
    private Long id;

    @Transient
    private Boolean delete;

    // remaining fields alphabetical
}
```

#### Repository (`repositories/clients/<Name>Repository.java`)

Simple (no images):
```java
public interface <Name>Repository extends CrudRepository<<Name>Entity, Long> {
}
```

With images — add `@Transactional`, `@Cacheable`, `@Query`, and `update` method. See `SoupRepository` for the pattern.

#### Converter (`converters/clients/<Name>Converter.java`)

Extends `ItemsConverter` (no images) or `ImageConverter` (with images). Must provide:
- `processItems(...)` — maps form params to entities
- `filterIdsToDelete(...)` — extracts IDs marked for deletion

With images, also provide:
- `filterItemsToInsert(...)` — items without an ID
- `filterItemsToUpdate(...)` — items with an ID

#### API Controller (`controllers/clients/<name>/<Name>ApiController.java`)

```java
@CrossOrigin("${app.clients.<CLIENT>.origin}")
@RestController
@RequiredArgsConstructor
@Slf4j
final class <Name>ApiController {
    private final GrafanaGateway grafanaGateway;

    private final <Name>Repository <name>Repository;

    @GetMapping("/api/<site-name>")
    List<<Name>Entity> getItems() {
        log.info("Incoming API request: <site-name>");
        grafanaGateway.registerRequest("<site-name>", "all");
        // fetch and return items
    }
}
```

#### Admin Controller (`controllers/clients/<name>/<Name>Controller.java`)

Delegate to the `admin-portal` skill for the full pattern.

### 6. Wire Into Existing Site Infrastructure (if new site)

If the client has its own site (not hosted under an existing domain), delegate to the `new-site` skill — it owns compose, Prometheus, build pipeline, and cache volume wiring.

### 7. Client Secret (WIP Sites Only)

If the site is a work-in-progress that needs cookie-based access control, see the `sites-wip` prompt — it owns the client secret mechanism and setup instructions.

## Naming Conventions

| Item | Format | Example |
|------|--------|---------|
| Authelia username | `info@<domain>` | `info@gaiapeeps.com` |
| App client key | lowercase short name | `peeps` |
| Env var | `APP_CLIENTS_<CLIENT>_TOKEN` | `APP_CLIENTS_PEEPS_TOKEN` |
| Caller ID | full site name | `gaiapeeps` |
| API path | `/api/<site-name>` | `/api/gaiapeeps` |
| View name | `clients/<domain-prefix>` | `clients/gaiapeeps` |
| Package | `controllers/clients/<CLIENT>` | `controllers/clients/peeps` |
| Table name | uppercase | `PEEPS` |

## Common Pitfall

If a user can authenticate via Authelia but sees a redirect loop or error page, the most likely cause is a missing Thymeleaf template for their email domain. The `ErrorController` logs `"Supressing Whitelabel Error Page"` and redirects to Authelia, creating an apparent loop.
