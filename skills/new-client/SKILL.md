---
name: new-client
description: Adding a new client to the platform. Use when onboarding a new user/site that needs Authelia login, API access, database storage, and an admin portal view.
---

# New Client Registration

## Overview

Adding a new client touches multiple systems. This skill covers the full wiring — from auth to API to admin portal. For database schema creation, delegate to the `liquibase` skill. For admin portal templates, delegate to the `admin-portal` skill.

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
| Prometheus | Scrape target (if new site) |

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
- Generate prod hash: `docker run --rm authelia/authelia:4.37 authelia crypto hash generate argon2 --password '<password>'`
- Test users all share the same hash (password: `test`): `$argon2id$v=19$m=65536,t=3,p=4$vSWXSYTyISnsOUqnNw6PQA$FvOvwV2ASM4Ii5ax2pt+lMAY+01DpZEtX+50IjPFhBw`

### 2. Application Config

Add client to `app.clients` in `application.yaml`:

```yaml
app:
  clients:
    <shortname>:
      caller: "<site-name>"
      origin: "https://<domain>"
      token: ${APP_CLIENTS_<UPPER>_TOKEN}
```

Add test override in `application-test.yaml`:

```yaml
app:
  clients:
    <shortname>:
      origin: "*"
      token: "test"
```

### 3. Environment Variables

Services `compose.yaml` — add to the `services` service environment:

```yaml
APP_CLIENTS_<UPPER>_TOKEN: $APP_CLIENTS_<UPPER>_TOKEN
```

Services `.env` — add the token value:

```
APP_CLIENTS_<UPPER>_TOKEN=<generated-token>
```

Sites `compose.yaml` — add `TOKEN` to the site service:

```yaml
environment:
  TOKEN: $APP_CLIENTS_<UPPER>_TOKEN
```

Sites `.env` — add the same token:

```
APP_CLIENTS_<UPPER>_TOKEN=<same-token>
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
@CrossOrigin("${app.clients.<shortname>.origin}")
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

If the client has its own site (not hosted under an existing domain), wire it into:

- `prometheus.yaml` — add `"sites_<site-name>"` to the `sites` job targets
- `prometheus-test.yaml` — add `"<site-name>"` to the `sites` job targets
- `sites/compose.yaml` — add service entry with Traefik labels and `TOKEN` env var
- `sites/compose-test.yaml` — add service entry with Traefik labels
- `sites/.github/workflows/build.yaml` — add `build-<name>` to `repository_dispatch.types` and `<name>` to the matrix

## Naming Conventions

| Item | Format | Example |
|------|--------|---------|
| Authelia username | `info@<domain>` | `info@gaiapeeps.com` |
| App client key | lowercase short name | `peeps` |
| Env var | `APP_CLIENTS_<UPPER>_TOKEN` | `APP_CLIENTS_PEEPS_TOKEN` |
| Caller ID | full site name | `gaiapeeps` |
| API path | `/api/<site-name>` | `/api/gaiapeeps` |
| View name | `clients/<domain-prefix>` | `clients/gaiapeeps` |
| Package | `controllers/clients/<shortname>` | `controllers/clients/peeps` |
| Table name | uppercase | `PEEPS` |

## Common Pitfall

If a user can authenticate via Authelia but sees a redirect loop or error page, the most likely cause is a missing Thymeleaf template for their email domain. The `ErrorController` logs `"Supressing Whitelabel Error Page"` and redirects to Authelia, creating an apparent loop.
