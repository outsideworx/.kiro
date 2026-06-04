# Python Utilities

## Overview

The `utils/` directory houses operational scripts and background services that run alongside the main application. These are generic utilities — not part of the Spring Boot app — deployed as a Python sidecar container.

## Runtime

- Image: `python:3.13-slim`
- Dependencies: `requirements.txt` (`psycopg2-binary`)
- Shared logging: `commons/logging_config.py`
- Deployed as a single container in the services stack

### Prod vs Test

| Aspect | Prod (compose.yaml) | Test (compose-test.yaml) |
|--------|---------------------|--------------------------|
| Script loading | Skips `*.draft.py` files | Runs ALL `*.py` files including drafts |
| DB credentials | `DB_USERNAME`/`DB_PASSWORD` from `.env` | `DB_USERNAME=postgres`, `DB_PASSWORD=""` |
| Volumes | Persistent host paths (`/home/outsideworx/utils`) | Local `./utils/cache` bind mount for cache output, `./utils` for scripts |
| ntfy volume | Host path `/home/outsideworx/ntfy` | Named volume `ntfy:` |

## Log Format

```
2025-05-11 13:00:00 INFO --- app=<name>: <message>
```

Set up via `setup_logging("app-name")` from `commons/logging_config.py`.

## Scripts

### cache.py — Image Cache Sync

Periodically syncs base64-encoded images from PostgreSQL to disk as JPEG files. The static sites serve these cached files instead of querying the DB on every request.

- Connects to PostgreSQL (env: `DB_USERNAME`, `DB_PASSWORD`, host `services_postgres:5432`)
- Polls every 60 seconds
- Tracks changes via a `hash` column on each table — only re-exports rows whose hash changed
- Persists known hashes to `/utils/cache/hashes.properties`
- Writes last successful scan time to `/utils/cache/last_scan.txt` (global) and `/utils/cache/<client>/last_scan.txt` (per-client)
- Output directories: `/utils/cache/ciafo/`, `/utils/cache/soup/`
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

`hashes.properties` uses dot-separated keys: `<client>.<itemId>=<hash>` (e.g., `ciafo.42=a1b2c3d4`).

## Operations Scripts

Shell scripts in `utils/operations/` — run manually on the server, not part of the sidecar.

### docker-stats.sh

Pretty-prints a table of all running containers with CPU %, RAM (MB), network I/O (GB), color-coded thresholds. Also checks for degraded Swarm services (replicas mismatch).

### docker-wipe.sh

Nuclear option: removes both stacks, all secrets, all images, prunes the system, deletes deployment directories and cache, then runs `apt upgrade`. Interactive confirmation required.

### system-swap.sh

Creates a 2GB swap file, persists it to fstab, tunes `vm.swappiness=10` and `vm.vfs_cache_pressure=50`. Idempotent — skips if swap already active.

## File Layout

```
services/utils/
├── cache.py                    # Image cache sync (active)
├── requirements.txt            # psycopg2-binary
├── commons/
│   └── logging_config.py       # Shared logging setup
└── operations/
    ├── docker-stats.sh         # Container resource overview
    ├── docker-wipe.sh          # Full system wipe
    └── system-swap.sh          # Swap file setup
```

## Environment Variables

| Variable | Used by | Description |
|----------|---------|-------------|
| `DB_USERNAME` | cache.py | PostgreSQL username (also used as DB name) |
| `DB_PASSWORD` | cache.py | PostgreSQL password |
