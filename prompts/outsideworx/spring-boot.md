# Spring Boot Application

## Stack

- Java 25, Spring Boot 3.5, Maven
- Spring Data JPA (Hibernate) + PostgreSQL 16
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
│   ├── IndexController.java         # OAuth2-protected portal (dispatches to ModelVisitor)
│   ├── ViewControllers.java         # Redirect mappings (/grafana, /ntfy)
│   ├── CallbackController.java      # Shared callback API
│   ├── ErrorController.java
│   ├── ModelVisitor.java            # Interface for client views
│   └── clients/
│       ├── ciafo/                   # CiafoApiController, CiafoController
│       ├── peeps/                   # PeepsApiController, PeepsController
│       └── soupart/ciafo/           # SoupApiController, SoupController
├── converters/
│   ├── ItemsConverter.java          # Base: items[n].field parsing
│   ├── ImageConverter.java          # Extends ItemsConverter: base64 + resize
│   └── clients/
│       ├── CiafoConverter.java      # Extends ImageConverter
│       ├── SoupConverter.java       # Extends ImageConverter
│       └── PeepsConverter.java      # Extends ItemsConverter (no images)
├── gateways/
│   ├── GrafanaGateway.java          # Micrometer counters
│   └── EmailGateway.java            # MailerSend
├── models/
│   ├── Callback.java                # DTO (@Data, no JPA)
│   ├── CallbackEntity.java          # JPA entity
│   └── clients/
│       ├── ciafo/
│       │   ├── CiafoEntity.java
│       │   └── mapping/             # Interface projections (CiafoPreview, CiafoPayload, etc.)
│       ├── peeps/PeepsEntity.java
│       └── soup/SoupEntity.java
└── repositories/
    ├── CallbackRepository.java
    └── clients/
        ├── CiafoRepository.java     # Custom @Query + @Cacheable
        ├── SoupRepository.java      # Custom @Query + @Cacheable
        └── PeepsRepository.java     # Plain CrudRepository
```

## Coding Conventions

### General Style

- All concrete classes are `final`
- Package-private visibility by default for controllers and filters (no `public` unless needed by framework)
- Converters and gateways are `public` (referenced cross-package)
- Entities and DTOs are `public` (referenced cross-package)
- Repositories are `public` (interface, referenced cross-package)
- One blank line between fields, no blank lines between annotations on the same element
- Annotations ordered alphabetically on classes (e.g., `@Component`, `@Order`, `@RequiredArgsConstructor`, `@Slf4j`)
- Fields ordered alphabetically within a class
- Imports: no wildcards, no static imports in production code (static imports allowed in tests)

### Lombok Usage

- `@Data` on entities and DTOs
- `@RequiredArgsConstructor` on all classes with injected dependencies (constructor injection only)
- `@Slf4j` on any class that logs
- No `@Builder`, `@AllArgsConstructor`, or `@Getter`/`@Setter` individually

### Dependency Injection

- Constructor injection exclusively (via `@RequiredArgsConstructor`)
- Dependencies declared as `private final` fields
- Exception: `@Value` for simple string properties (e.g., `WebSecurity.logoutUrl`)

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
- `CacheFilter` adds `Cache-Control: public, max-age=86400` header for `/api/cached/**` requests

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
- Log format in code: `log.info("Message: [{}]", value)` — square brackets around interpolated values

### Database / Liquibase

- Changelog master: `db/changelog-master.yaml` with `include` entries
- Naming: `NNN-description.yaml` (e.g., `000-create-ciafo.yaml`, `001-hash-trigger-ciafo.yaml`)
- Author: `services`
- Changes use raw SQL (`- sql: sql: |`)
- Sequences: `<table>_seq START WITH 1 INCREMENT BY 50`
- Tables: lowercase names, `BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY`

## Testing Conventions

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

- Class annotations (in order): `@ActiveProfiles("test")`, `@AutoConfigureMockMvc` (if MockMvc), `@Import(IntegrationTestBase.class)`, `@RequiredArgsConstructor`, `@SpringBootTest(classes = SpringApplication.class)`
- Constructor injection for test dependencies (enabled by `spring.test.constructor.autowire.mode=all`)
- `@BeforeEach` to clean DB state (`repository.deleteAll()`)
- Suffix: `IT` (e.g., `PeepsControllerIT`)
- Package: `it.controllers`, `it.repositories`, `it.configuration`
- Run via Maven Failsafe plugin (`mvn verify`)

### Unit Tests

- `@ExtendWith(MockitoExtension.class)`
- `@Mock` for dependencies, `@InjectMocks` for subject
- Suffix: `Test` (e.g., `AuthTokenFilterTest`)
- Package mirrors production: `net.outsideworx.services.configuration`
- Run via Maven Surefire plugin (`mvn test`)

### Assertions

- AssertJ exclusively (`assertThat(...)`)
- `assertThatThrownBy(...)` for exception testing
- MockMvc: `status()`, `content()`, `jsonPath()`, `redirectedUrl()`
- Mockito: `verify(...)`, `verifyNoInteractions(...)`, `when(...).thenReturn(...)`

### Test Naming

- `methodName_whenCondition_expectedBehavior` (e.g., `getItems_withValidCredentials_returnsOk`)
- Helper methods: `entity(...)` factory methods for creating test data

## Prod vs Test

| Aspect | Prod (default profile) | Test (`-test` profile) |
|--------|------------------------|------------------------|
| Port | 80 (app), 81 (actuator) | 8080 |
| Cache | Caffeine (`maximumSize=76`) | Disabled (`type: none`) |
| Datasource | `jdbc:postgresql://postgres:5432/` with credentials | `jdbc:postgresql://localhost:5432/` trust auth |
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
- Docker image: `FROM openjdk:25-ea`, `java -jar services.jar`
