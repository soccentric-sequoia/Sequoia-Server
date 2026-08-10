# Sequoia Server

The **Sequoia Server** is the central orchestration and data-collection backend for the Sequoia IV&V (Independent Validation & Verification) tool suite. It exposes:

- A **gRPC service** (`HalService`) for the embedded HAL Client App to submit test results and receive streaming commands.
- A **REST/HTTP API** (`/api/v1`) for the Web UI and external tooling.
- A **PostgreSQL** backend with versioned migrations.
- **OTA firmware upgrade** orchestration.
- **Prometheus metrics** (optional).

---

## 🚀 Quick Start (Docker — Recommended)

**The easiest and most reliable way** to get started on any machine:

```bash
cd docker
./setup.sh
```

This will:
1. Check Docker prerequisites
2. Create `.env` from `.env.example`
3. Build the Docker image for your platform (auto-detected)
4. Start PostgreSQL and dev environment
5. You're ready to build!

Then enter the container and build:
```bash
docker exec -it sequoia-dev bash
cmake --preset debug && cmake --build build/debug
```

### Alternative: VS Code Dev Container

Open this project in VS Code and:
1. Install the "Dev Containers" extension
2. Press `F1` → "Dev Containers: Reopen in Container"
3. Wait for build (first time takes ~10 min)

Everything is configured automatically!

See [docker/README.md](docker/README.md) for detailed Docker documentation.

---

## Architecture Overview

```
HAL Client (embedded)  ──gRPC (port 9090)──►  HalServiceImpl
                                                    │
Web UI / REST clients  ──HTTP (port 8080)──►  Drogon REST API
                                                    │
                                         ┌──────────┴──────────┐
                                         │   Service Layer      │
                                         │  RunService          │
                                         │  LogService          │
                                         │  AuthService         │
                                         │  OtaService  ...     │
                                         └──────────┬──────────┘
                                                    │
                                         ┌──────────┴──────────┐
                                         │   Repository Layer   │
                                         │  (Drogon ORM / PG)  │
                                         └──────────┬──────────┘
                                                    │
                                              PostgreSQL 15+
```

---

## Prerequisites (Native Build)

**Note:** Using Docker (see Quick Start above) avoids all manual dependency installation.

For native builds without Docker:

| Tool | Version |
|------|---------|
| CMake | 3.22+ |
| vcpkg | latest |
| C++ compiler | GCC 12+ or Clang 16+ |
| PostgreSQL | 15+ |
| protoc + grpc_cpp_plugin | matching vcpkg grpc version |

Set `VCPKG_ROOT` in your environment before building.

---

## Quick Start (Local Dev)

```bash
cp .env.example .env
# Edit .env — at minimum set SEQUOIA_DB_* and SEQUOIA_SESSION_SECRET

# Start Postgres
docker compose -f docker/docker-compose.yml up postgres -d

# Build and run (loads .env automatically)
./scripts/run_dev.sh
```

Or with Docker Compose (full stack):

```bash
docker compose -f docker/docker-compose.yml up --build
```

---

## Building

```bash
# Debug (default dev preset)
cmake --preset debug
cmake --build build/debug

# Release
cmake --preset release
cmake --build build/release

# Run tests
cmake --preset test
cmake --build build/test
ctest --test-dir build/test --output-on-failure

# ASan
cmake --preset asan
cmake --build build/asan
```

---

## Configuration

All configuration is via environment variables. See `.env.example` for the full list.

| Variable | Default | Description |
|----------|---------|-------------|
| `SEQUOIA_GRPC_LISTEN` | `0.0.0.0:9090` | gRPC bind address |
| `SEQUOIA_HTTP_LISTEN` | `0.0.0.0:8080` | HTTP bind address |
| `SEQUOIA_DB_HOST` | `localhost` | PostgreSQL host |
| `SEQUOIA_DB_NAME` | `sequoia` | Database name |
| `SEQUOIA_DB_USER` | `sequoia` | DB username |
| `SEQUOIA_DB_PASSWORD` | *(required)* | DB password |
| `SEQUOIA_SESSION_SECRET` | *(required, ≥32 chars)* | Session signing key |
| `SEQUOIA_REGISTRY_PATH` | `registry.yaml` | Module registry YAML |
| `SEQUOIA_OTA_STORAGE_PATH` | `/var/sequoia/ota` | OTA artifact directory |
| `SEQUOIA_LOG_LEVEL` | `info` | Log level (trace/debug/info/warn/error) |

---

## Module Registry

Modules are declared in a YAML file (path set by `SEQUOIA_REGISTRY_PATH`):

```yaml
modules:
  - id: "com.example.temperature"
    name: "Temperature Monitor"
    description: "Reads temperature from sensor A"
    version: "1.2.0"
    result_type_url: "type.googleapis.com/sequoia.TempResult"
    config_type_url: "type.googleapis.com/sequoia.TempConfig"
    modes: ["single_shot", "monitoring"]
```

---

## Database Migrations

SQL migration files live in `migrations/`. They are applied automatically at startup in alphabetical order. Migrations are idempotent (tracked in `schema_migrations`). To add a new migration, create `migrations/NNNN_description.sql`.

---

## API

All protected endpoints require either:
- A session cookie (from `POST /api/v1/auth/login`), or
- An `Authorization: Bearer <token>` header.

### Key endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/health` | Health check |
| POST | `/api/v1/auth/login` | Login → sets session |
| POST | `/api/v1/auth/tokens` | Create API token |
| GET | `/api/v1/modules` | List test modules |
| POST | `/api/v1/runs` | Start a test run |
| GET | `/api/v1/runs/{id}` | Get run details |
| GET | `/api/v1/runs/{id}/logs` | List run logs |
| GET | `/api/v1/runs/{id}/logs/tail` | SSE live log tail |
| GET | `/api/v1/target` | Get board info |
| POST | `/api/v1/target/heartbeat` | Board heartbeat |
| POST | `/api/v1/ota/upload` | Upload OTA artifact |
| POST | `/api/v1/ota/{id}/trigger` | Trigger OTA upgrade |

---

## Decisions Log

| Decision | Rationale |
|----------|-----------|
| gRPC sync API for unary, streaming wait-loop | Simpler than async CQ for initial impl; async CQ adds complexity without benefit at this scale |
| Drogon for REST | Built-in PG ORM, async, WebSocket, SSE, session management |
| argon2id for passwords | OWASP recommended |
| SHA-256 for API tokens | Tokens are long random strings; SHA-256 is fast and sufficient |
| 1-minute minimum cron resolution | Full cron parser adds a dependency; document the limitation |
| Single `board` row (unique partial index) | The system targets exactly one physical test board |
| Migrations at startup | Ensures schema is always up to date; no separate migration tool needed |
| vcpkg manifest mode | Reproducible dependency pinning per repo |

## Build

```sh
cmake -B build -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build build -j$(nproc)
```

Binary: `build/sequoia-server`

## Run (dev)

```sh
./build/sequoia-server --config etc/sequoia/server.toml --log-level debug
```

Without a config file the binary starts with compiled-in defaults.

## Test

```sh
cmake --build build --target config-test
ctest --test-dir build --output-on-failure
```

## Config

Default path: `/etc/sequoia/server.toml`. See `etc/sequoia/server.toml` for all keys and defaults.

CLI flags override file values:

| Flag | Default |
|------|---------|
| `--config PATH` | `/etc/sequoia/server.toml` |
| `--log-level LEVEL` | `info` |

## Implementation Status

| Step | Status |
|------|--------|
| 1 · Skeleton + config + logger | ✓ done |
| 2 · DB layer + migrations | — |
| 3 · DB CRUD (runs + batches) | — |
| 4 · Ingest queue + result writer | — |
| 5 · gRPC bidi server | — |
| … | — |
