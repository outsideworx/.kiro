# Python Utilities

## Overview

The `utils/` directory houses operational scripts and background services that run alongside the main application. These are generic utilities — not part of the Spring Boot app — deployed as a Python sidecar container.

## Runtime

- Image: `python` (slim variant)
- Dependencies: `requirements.txt` (`psycopg2-binary`)
- Shared logging: `commons/logging_config.py`
- Deployed as a single container in the services stack

### Script Delivery

- **Prod**: scripts are **not** bind-mounted. `deploy.sh` packs the `utils/` folder into a single deterministic `utils.tar`, injected as one Swarm config named `utils` and mounted at `/etc/utils.tar`. The container extracts it to `/etc/utils` at startup, then runs every top-level `*.py` (skipping `*.draft.py`). A tarball is used because Swarm configs are single files but the sidecar needs a directory tree (`cache.py` + the `commons/` package + `requirements.txt`). Adding a new script needs no compose/`deploy.sh` change — it is picked up from the tar automatically. See `services-deployment.md` → "The `utils` Config Tarball". This replaced the former `./utils:/etc/utils` bind mount, which failed when the Docker daemon did not share the repo's host filesystem (`bind source path does not exist`).
- **Test**: still a plain `./utils:/etc/utils` bind mount (every test service uses bind mounts, and test deliberately runs ALL `*.py` including drafts for live editing). The bind works on a normal dev machine running `docker compose up`.

### Prod vs Test

| Aspect | Prod (compose.yaml) | Test (compose-test.yaml) |
|--------|---------------------|--------------------------|
| Script loading | Skips `*.draft.py` files | Runs ALL `*.py` files including drafts |
| DB credentials | `DB_USERNAME`/`DB_PASSWORD` from `.env` | `DB_USERNAME=postgres`, `DB_PASSWORD=""` |
| DB host | `DB_HOST=services_postgres` | `DB_HOST=postgres` |
| Script source | `utils.tar` Swarm config, extracted to `/etc/utils` at startup | `./utils` bind mount at `/etc/utils` |
| Volumes | Named volume `cache:` for cache output (exposed to sites as `services_cache`) | Named volume `cache:` for cache output (shared with sites as `services_cache`), `./utils` bind mount for scripts |

## Log Format

```
2025-05-11 13:00:00 INFO --- app=<name>: <message>
```

Set up via `setup_logging("app-name")` from `commons/logging_config.py`.

## Scripts

### cache.py — Image Cache Sync

Periodically syncs base64-encoded images from PostgreSQL to disk as JPEG files. The static sites serve these cached files instead of querying the DB on every request.

- Connects to PostgreSQL (env: `DB_HOST`, `DB_USERNAME`, `DB_PASSWORD`)
- Polls every 60 seconds
- Tracks changes via a `hash` column on each table — only re-exports rows whose hash changed
- Persists known hashes to `/utils/cache/hashes.properties`
- Writes last successful scan time to `/utils/cache/<site>/last_scan.txt` (per-site)
- Output directories: `/utils/cache/come-in-and-find-out/`, `/utils/cache/soupart/` (named after the site repo, not the DB table `ciafo`/`soup`)
- File naming: `<categoryId>_<itemId>_<label>.jpg` (e.g., `3_42_thumbnail1.jpg`)
- Category index: `categories.properties` per client with category names and ordered item IDs
- Handles table-not-found gracefully (logs error, continues)
- Cleans up files for deleted rows

#### Category Index Format

Each client directory contains a `categories.properties` file:

```properties
1=Accessories
1.ids=5,12,18
2=Art
2.ids=3,7,22
3=Furniture
3.ids=1,9,15,20,25,30,35
```

Categories are sorted alphabetically and assigned numeric IDs starting at 1. The `.ids` key lists item IDs in database order (by primary key ascending). These numeric category IDs are used as the first segment of image filenames.

#### Hashes Format

`hashes.properties` uses dot-separated keys: `<site>.<itemId>=<hash>` (e.g., `come-in-and-find-out.42=a1b2c3d4`). The prefix is the site-name subdir, not the DB table name.

## File Layout

```
services/utils/
├── cache.py                    # Image cache sync (active)
├── requirements.txt            # psycopg2-binary
└── commons/
    └── logging_config.py       # Shared logging setup
```

## Environment Variables

| Variable | Used by | Description |
|----------|---------|-------------|
| `DB_HOST` | cache.py | PostgreSQL host (`services_postgres` in prod, `postgres` in test) |
| `DB_PASSWORD` | cache.py | PostgreSQL password |
| `DB_USERNAME` | cache.py | PostgreSQL username (also used as DB name) |
