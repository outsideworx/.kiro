# Coding Conventions

## Universal Rule

All lists, keys, and identifiers are **alphabetical** unless a stronger ordering rule applies (noted below).

Stronger ordering rules:
- **Visibility-descending**: `public` before `private`, broader scope before narrower scope
- **Logical flow**: callers before callees, setup before execution
- **Convention-fixed**: positions dictated by a framework or format (e.g., GitHub Actions top-level keys)

"Semantic importance" is never a reason to deviate — a key that feels like a mode declaration or top-level setting still sorts alphabetically with the rest.

## Java

### Class-Level

- All concrete classes are `final`
- Exception: `@Configuration` and `@ConfigurationProperties` classes are not `final` (CGLIB proxying requires subclassing)
- Visibility: package-private by default (controllers, filters); `public` only when required cross-package (converters, gateways, entities, repositories, interfaces)
- Annotations on classes: alphabetical (e.g., `@Component`, `@Order`, `@RequiredArgsConstructor`, `@Slf4j`)
- No blank lines between annotations on the same element

### Imports

Order by group, alphabetical within each group:
1. Project imports (`net.outsideworx.services.*`)
2. Third-party (`com.*`, `io.*`, `jakarta.*`, `lombok.*`, `org.*`)
3. Java standard library (`java.*`)

Rules:
- No wildcard imports in production code
- No static imports in production code
- Static imports allowed in tests (AssertJ, Mockito, MockMvc matchers)

### Fields

- `private final` dependencies first (alphabetical)
- One blank line between each field declaration
- Entity fields: `@Id @GeneratedValue` first, then remaining fields alphabetical
- `@Transient` fields sort alphabetically with the rest (annotation stays on the field)

### Methods

Ordering hierarchy:
1. `public` methods before `private` methods
2. Within same visibility: logical flow (called methods appear after their callers)

### Dependency Injection

- Constructor injection exclusively via `@RequiredArgsConstructor`
- Exception: `@Value` for simple config strings that don't warrant a `Properties` entry

### Lombok

- `@Data` on entities and DTOs
- `@RequiredArgsConstructor` on all classes with injected dependencies
- `@Slf4j` on any class that logs
- No `@Builder`, `@AllArgsConstructor`, or `@Getter`/`@Setter` individually

### Naming

- Test methods: `methodName_whenCondition_expectedBehavior`
- Helper methods in tests: `entity(...)` factory pattern
- Entity table names: uppercase (`@Table(name = "CIAFO")`)
- SQL column/table references in queries: uppercase table, lowercase columns

## YAML — Docker Compose

### Top-Level Keys

Alphabetical: `configs`, `networks`, `secrets`, `services`, `volumes`

### Services

- Listed alphabetically
- Keys within each service alphabetical: `command`, `configs`, `deploy`, `environment`, `image`, `networks`, `ports`, `secrets`, `user`, `volumes`
- Within `deploy`: `labels`, `mode`, `placement` (alphabetical)
- Within `placement`: `constraints`

### Labels (Traefik)

- List format (`- key=value`), alphabetical by full label path
- Order: `traefik.enable` → `traefik.http.middlewares.*` → `traefik.http.routers.*` → `traefik.http.services.*`
- Within routers: `entrypoints`, `middlewares`, `rule`, `tls.*`
- Within services: `loadbalancer.healthcheck.*`, `loadbalancer.server.port`

### Environment

- Map format (`KEY: value`), keys alphabetical

### Volumes (within a service)

- List format, alphabetical by host path

## YAML — Application Config

- Top-level keys alphabetical: `app`, `management`, `server`, `spring`
- Nested keys alphabetical at every level
- Named entries (clients, services) alphabetical: `ciafo`, `peeps`, `soup`

## YAML — GitHub Actions

- Top-level keys follow GHA convention: `name`, `on`, `jobs` (not alphabetical)
- `workflow_dispatch` inputs: alphabetical
- `with` blocks: keys alphabetical
- Steps: logical order (checkout → setup → build → deploy)

## Maven (pom.xml)

### Dependencies

Grouped by purpose with XML comments as section headers:
1. `<!-- Libraries -->` — Spring Boot starters (alphabetical by artifactId)
2. `<!-- Test -->` — test-scoped dependencies (alphabetical)
3. `<!-- Connector -->` — runtime connectors and SDKs (alphabetical)
4. `<!-- Analytics -->` — metrics
5. `<!-- Utilities -->` — utility libraries (alphabetical)

Within each group: alphabetical by `artifactId`.

### Dependency XML Element Order

`artifactId` before `groupId` (reverse of Maven convention — project-specific choice).

### Plugins

Alphabetical by `artifactId`: `maven-compiler-plugin`, `maven-failsafe-plugin`, `spring-boot-maven-plugin`.

## Tests

### Unit Tests

- `@ExtendWith(MockitoExtension.class)`
- `@Mock` fields alphabetical, then `@InjectMocks` last
- One blank line between fields
- Suffix: `Test`

### Integration Tests

- Annotations alphabetical: `@ActiveProfiles("test")`, `@AutoConfigureMockMvc`, `@Import(IntegrationTestBase.class)`, `@RequiredArgsConstructor`, `@SpringBootTest(classes = SpringApplication.class)`
- Constructor injection for test dependencies
- `@BeforeEach` for DB cleanup (`repository.deleteAll()`)
- Helper methods (`entity(...)`) before or after test methods
- Suffix: `IT`

### Assertions

- AssertJ only (`assertThat`, `assertThatThrownBy`)
- MockMvc: `status()`, `content()`, `jsonPath()`, `redirectedUrl()`
- Mockito: `verify`, `verifyNoInteractions`, `when(...).thenReturn(...)`

## Python

### File Structure

- Imports at top: stdlib first, then third-party, then project (`commons.*`)
- Module-level constants after imports, uppercase with underscores (`DB_HOST`, `INTERVAL`, `OUTPUT_DIR`)
- Constants alphabetical when they form a logical group (e.g., `CIAFO_LABELS`, `HASH_FILE`, `SCAN_TIME_FILE`, `SOUP_LABELS`)
- `setup_logging("<app-name>")` call immediately after imports
- Functions defined after constants
- Main loop / `web.run_app(...)` at the bottom of the file (no `if __name__ == "__main__"` guard)

### Style

- No classes — functional/procedural style with module-level functions
- f-strings for string formatting
- `logging.info(...)` / `logging.error(...)` directly (no `log` variable)
- Exception handling: catch specific exceptions, log the error, continue where possible
- File I/O: `with open(...)` context managers
- No type annotations
- No docstrings — code is self-documenting with clear function names

### Naming

- Functions: `snake_case`, verb-first (`load_hashes`, `sync_images_to_disk`, `save_hashes`)
- Constants: `UPPER_SNAKE_CASE`
- Local variables: `snake_case`, short but descriptive (`cur`, `conn`, `hashes`)
- Loop variables: single letter or short (`id_`, `hash_`, `row`)
- Trailing underscore to avoid shadowing builtins (`id_`)

### Dependencies

- `requirements.txt`: one package per line, alphabetical, no version pins
- Prefer lightweight packages (`psycopg2-binary` over SQLAlchemy, `aiohttp` for async HTTP)

### Logging Format

```python
setup_logging("app-name")
# Produces: 2025-05-11 13:00:00 INFO --- app=<name>: <message>
```

- Log level and app name in every line
- f-strings in log messages (not `%s` formatting)

### Async (aiohttp)

- `web.Application()` with `on_startup` / `on_cleanup` hooks for session lifecycle
- Catch-all route: `app.router.add_route("*", "/{path_info:.*}", handler)`
- Header filtering via set comprehension: `{k: v for k, v in headers.items() if k.lower() not in SKIP_SET}`
- Skip-header sets defined as module-level constants

## Bash

### Script Types

Two categories with different conventions:

1. **Deployment scripts** (`deploy.sh`) — `#!/bin/bash`, `set -e` after variable declarations
2. **Operations scripts** (`operations/*.sh`) — `#!/bin/bash` or `#!/usr/bin/env bash`, `set -euo pipefail` for strict mode

### Structure

- Shebang line first
- Constants/variables at top (uppercase)
- `set -e` (or `set -euo pipefail`) after initial variable declarations
- Flag handling via `if [ "$1" == "--flag" ]` blocks with `exit 0` after each
- Unknown flag guard: `if [ -n "$1" ]; then echo "Error: ..."; exit 1; fi`
- Main logic after all flag handling

### Style

- Double quotes around all variable expansions (`"$VAR"`, `"$1"`)
- `[ ... ]` for test expressions (not `[[ ... ]]` in deploy scripts; `[[ ... ]]` allowed in operations scripts with `bash`)
- `$(command)` for command substitution (not backticks)
- No function keyword: `function_name() { ... }` (only in docker-stats.sh for utilities)
- `echo` for simple output, `printf` for formatted output
- Color codes as uppercase constants (`RED`, `GREEN`, `BOLD`, `RESET`)
- `awk "BEGIN {...}"` for arithmetic (not `bc` or `$((...))` for floats)

### Naming

- Script files: `kebab-case.sh`
- Variables: `UPPER_SNAKE_CASE` for constants, `lower_snake` for locals
- No exported variables unless needed by child processes

### Patterns

- Idempotent operations: check before acting (`if swapon --show | grep -q ...`)
- Interactive confirmation: `read -p "..." ans; [[ $ans =~ ^[Yy]$ ]] || { echo "Aborted."; return; }`
- `cp` with multiple source files on one line (alphabetical)
- `mkdir -p` for directory creation (no error if exists)
- Docker commands: `docker stack deploy -c compose.yaml <stack> --detach=false --resolve-image=always`
- Force-update pattern: `docker stack services <stack> --format '{{.Name}}' | xargs -I{} docker service update --force {}`

## General Style

- No trailing whitespace
- Single trailing newline at end of file
- 2-space indent for YAML
- 4-space indent for Java and Python
- Strings in YAML: quoted when containing special characters, unquoted for simple values
- Log messages: `log.info("Message: [{}]", value)` — square brackets around interpolated values (Java)
- Exception messages: sentence case, end with period
