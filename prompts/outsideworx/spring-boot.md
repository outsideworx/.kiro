# Spring Boot Application

## Stack

- Java, Spring Boot, Maven
- Spring Data JPA (Hibernate) + PostgreSQL
- Liquibase (YAML changelogs)
- Lombok (`@Data`, `@RequiredArgsConstructor`, `@Slf4j`)
- Caffeine cache
- Micrometer + Prometheus metrics
- Thymeleaf (admin portal templates)
- MailerSend SDK (email gateway)
- Thumbnailator (image resizing)
- Testcontainers + JUnit 5 + MockMvc + Mockito + AssertJ

## Package Structure

```
net.outsideworx.services/
├── SpringApplication.java
├── configuration/
│   ├── utils/
│   │   ├── Properties.java          # @ConfigurationProperties(prefix = "app")
│   │   └── FilterConditions.java    # Reusable request predicates (see auth.md)
│   ├── WebSecurity.java             # See auth.md
│   ├── AuthTokenFilter.java         # See auth.md
│   ├── CacheFilter.java
│   └── LogbackFilter.java
├── controllers/
│   ├── IndexController.java         # @RestController, OAuth2-protected portal (dispatches to ModelVisitor)
│   ├── ViewControllers.java         # Redirect mappings (/grafana, /ntfy)
│   ├── CallbackController.java      # Shared callback API
│   ├── ErrorController.java
│   ├── ModelVisitor.java            # Interface for client views
│   └── clients/<name>/              # <Name>ApiController, <Name>Controller per client
├── converters/
│   ├── ItemsConverter.java          # Base: items[n].field parsing
│   ├── ImageConverter.java          # Extends ItemsConverter: base64 + resize
│   └── clients/                     # <Name>Converter per client
├── gateways/
│   ├── GrafanaGateway.java          # Micrometer counters
│   └── EmailGateway.java            # MailerSend
├── models/
│   ├── Callback.java                # DTO (@Data, no JPA)
│   ├── CallbackEntity.java          # JPA entity
│   └── clients/<name>/              # <Name>Entity per client (+ mapping/ interfaces if needed)
└── repositories/
    ├── CallbackRepository.java
    └── clients/                     # <Name>Repository per client
```

## Coding Conventions

For Java coding conventions (class-level rules, imports, fields, dependency injection, Lombok, naming), see `coding-conventions.md`.

### Controllers

- API controllers: `@RestController`, `final`, package-private
- Admin controllers: `@Controller`, package-private, implement `ModelVisitor`
- Return types: `List<T>` for API endpoints, `String` (redirect) or `ModelAndView` for admin
- Log every incoming request with `log.info`
- Register metrics via `GrafanaGateway.registerRequest(endpoint, fetch)`

### The Client Pattern

Each client (ciafo, peeps, soup) follows this structure:

1. **Entity** — `@Data @Entity @Table(name = "UPPER_CASE")`, `@Id @GeneratedValue`, `@Transient Boolean delete` field for form handling
2. **Mapping interfaces** — Interface-based projections for partial selects (only for clients with complex queries)
3. **Repository** — Extends `CrudRepository<Entity, Long>`, `@Transactional` on interface
4. **Converter** — Extends `ItemsConverter` or `ImageConverter`, `@Component`, provides `processItems(...)`, `filterItemsToInsert(...)`, `filterItemsToUpdate(...)`, `filterIdsToDelete(...)`
5. **ApiController** — `@RestController`, serves JSON to the static site frontend
6. **Controller** — `@Controller`, implements `ModelVisitor`, handles form POST from admin portal, returns `"redirect:/"`

### Form Data Convention (items[n].field)

- HTML forms submit fields as `items[0].title`, `items[0].link`, `items[1].title`, etc.
- `ItemsConverter.getIterators(params)` extracts distinct indices via regex
- `ItemsConverter.getValue(params, iterator, field)` retrieves a specific field value
- File uploads use the same pattern: `items[0].image1` as `MultipartFile`
- The `delete` field is a checkbox (`"on"` = true)
- Empty/whitespace values are treated as null via `StringUtils.isEmptyOrWhitespace` filter

### Image Handling

- `ImageConverter.getImage(files, iterator, field)` → resizes to 1920×1080, encodes as base64 JPEG with `data:image/jpeg;base64,` prefix
- `ImageConverter.getThumbnail(files, iterator, field)` → resizes to 480×360, same encoding
- Images stored as `TEXT` columns in PostgreSQL (base64-encoded)
- Null returned for empty uploads (preserves existing value via `COALESCE` in update queries)

### Repositories

- `@Transactional` on the interface level
- Native queries (`nativeQuery = true`) with `@Query`
- `@Cacheable` on read methods, `@CacheEvict(allEntries = true)` on the controller POST method
- Custom `update` method using `@Modifying @Query` with SpEL (`:#{}`) for entity field access
- `COALESCE` pattern: `COALESCE(:#{#item.field}, field)` — only overwrite if new value is non-null

### Caching

- Caffeine with `maximumSize=76`
- Cache names: `ciafoItems`, `soupItems`
- Cache keys: category, id, or `category + offset`
- Eviction: `@CacheEvict(value = "...", allEntries = true)` on POST endpoints
- `CacheFilter` adds `Cache-Control: public, max-age=86400` header for `/api/cache/**` requests

### Filters (HttpFilter)

- Extend `jakarta.servlet.http.HttpFilter`
- Use `FilterConditions` for request matching predicates
- `LogbackFilter` — highest precedence, sets MDC `requestId` from `X-Request-Id` header (or generates base64 UUID fallback)
- `AuthTokenFilter` — see auth.md
- `CacheFilter` — sets cache headers for cached API responses

### Configuration

- `Properties` class maps `app.clients` and `app.services` from YAML
- Each client has: `caller`, `origin`, `token` (see auth.md for details)
- Each service has: `url` (external URL for redirects)
- Profiles: default (prod) and `test`

### Logging

- Logback pattern: `%level %logger{36} --- requestId=%X{requestId:-unknown}: %msg%n`
- MDC key: `requestId` (from X-Request-Id header or generated)

## Testing Conventions

For test conventions (annotations, naming, assertions, field ordering), see `coding-conventions.md`.

### Test Structure

```
src/test/java/
├── it/                              # Integration tests (IT suffix)
│   ├── IntegrationTestBase.java     # @TestConfiguration with Testcontainers PostgreSQL
│   ├── configuration/               # Filter/security ITs
│   ├── controllers/                 # Controller ITs (MockMvc)
│   └── repositories/                # Repository ITs
└── net/outsideworx/services/        # Unit tests (Test suffix)
    ├── configuration/               # Filter unit tests
    ├── converters/                   # Converter unit tests
    └── gateways/                     # Gateway unit tests
```

### Integration Tests

- Constructor injection for test dependencies (enabled by `spring.test.constructor.autowire.mode=all`)
- `IntegrationTestBase` provides a shared Testcontainers PostgreSQL instance
- Package: `it.controllers`, `it.repositories`, `it.configuration`
- Run via Maven Failsafe plugin (`mvn verify`)

### Unit Tests

- Package mirrors production: `net.outsideworx.services.configuration`
- Run via Maven Surefire plugin (`mvn test`)

## Prod vs Test

| Aspect | Prod (default profile) | Test (`-test` profile) |
|--------|------------------------|------------------------|
| Port | 80 (app), 81 (actuator) | 8080 |
| Cache | Caffeine (`maximumSize=76`) | Disabled (`type: none`) |
| Datasource | `jdbc:postgresql://services-postgres:5432/` with credentials | `jdbc:postgresql://localhost:5432/` trust auth |
| SSL | Normal validation | Trust-all (via `CommandLineRunner` in `@Profile("test")`) |
| OAuth URLs | `https://oauth.outsideworx.net/...` | `https://oauth.localhost/...` |
| Client tokens | From environment variables | Hardcoded `"test"` |
| Client origins | Specific domains | `"*"` (permissive CORS) |
| Service URLs | `https://grafana.outsideworx.net` etc. | `https://grafana.localhost` etc. |

The test profile is activated by `@ActiveProfiles("test")` in integration tests and by the `compose-test.yaml` environment. Config lives in `application-test.yaml`.

## Build & Run

- Build: `mvn package -DskipTests`
- Unit tests: `mvn test`
- Full verify (unit + integration): `mvn verify`
- Prod port: 80 (app), 81 (actuator/metrics)
- Test port: 8080
- Docker image: `FROM openjdk`, `java -jar services.jar`
