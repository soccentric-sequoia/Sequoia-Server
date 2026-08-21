# Sequoia-Server

## Project Summary

Sequoia-Server is the central orchestration and data-collection backend for the
Sequoia IV&V tool suite. The C++ service build is configured with CMake presets,
Docker development assets, Doxygen, and a vcpkg dependency manifest. The
documented architecture exposes a gRPC HalService for embedded clients and a
Drogon REST API for web and external tooling. Service-layer responsibilities
include runs, logs, authentication, OTA orchestration, health, modules, target
heartbeats, and live log streaming. PostgreSQL persistence is represented by
repositories and versioned SQL migrations; configuration is read from
environment variables and an optional TOML file, with CLI log-level/config
overrides. Optional Prometheus metrics and Docker Compose support are also
documented.

The repository contains unit tests for registry, gRPC backpressure, services,
time/UUID/hash/result conversion, plus a migration integration test and a
configuration test. These are useful contract evidence, but passing tests do
not prove a deployed PostgreSQL instance, authenticated web client, embedded
HAL client, OTA artifact store, or production TLS/network configuration. The
README therefore keeps the useful quick-start/build/API material while
marking environment-dependent features as partial or unknown. Use the checked-
in CMake presets and Docker setup for reproducible local validation, and treat
the `.env.example` secret fields as required deployment inputs.

The API and migration inventories are intentionally concise: route presence
does not establish authorization correctness, durable persistence, or a
successful client handshake. Docker setup can provide local dependencies, but
it does not substitute for production secrets, backups, observability, or
failure-recovery exercises. Keep endpoint behavior aligned with the Dashboard,
Client, and PySDK contracts when changing this service.

## Feature Inventory

| # | Name / Description | Implemented? | Test Coverage? | Manual Test |
|---:|---|---|---|---|
| 1 | CMake service build and debug/release/test presets | Implemented | Build/test commands present | [ ] Manual verification required |
| 2 | gRPC HalService client ingestion and command stream | Implemented | Backpressure test present | [ ] Manual verification required |
| 3 | REST `/api/v1` service surface | Implemented | Service tests present | [ ] Manual verification required |
| 4 | Authentication, sessions, and API tokens | Partial | Auth-related service tests present | [ ] Manual verification required |
| 5 | Module registry and module listing | Implemented | `tests/unit/registry/ModuleRegistryTest.cc` | [ ] Manual verification required |
| 6 | Run orchestration and result conversion | Implemented | Run/result unit tests present | [ ] Manual verification required |
| 7 | Log storage, listing, and SSE tail | Partial | Focused coverage not complete | [ ] Manual verification required |
| 8 | PostgreSQL repositories and startup migrations | Partial | Migration integration/config tests present | [ ] Manual verification required |
| 9 | Target heartbeat and board information | Implemented | Service coverage partial | [ ] Manual verification required |
| 10 | OTA upload and trigger orchestration | Partial | No complete device rollout test found | [ ] Manual verification required |
| 11 | Prometheus metrics | Partial | No focused metrics test found | [ ] Manual verification required |
| 12 | Docker Compose development environment | Implemented | Environment build only | [ ] Manual verification required |
| 13 | Production deployment, secrets, and TLS | Unknown | No deployment verification found | [ ] Manual verification required |

## Build and Test

```bash
cmake --preset debug && cmake --build build/debug
cmake --preset test && cmake --build build/test
ctest --test-dir build/test --output-on-failure
```

See [LOCAL_SETUP.md](LOCAL_SETUP.md), [docker/README.md](docker/README.md),
and `.env.example` for environment setup. Test sources are under `tests/` and
`test/`.
- A **REST/HTTP API** (`/api/v1`) for the Web UI and external tooling.
- A **PostgreSQL** backend with versioned migrations.
- **OTA firmware upgrade** orchestration.
- **Prometheus metrics** (optional). --- ## 🚀 Quick Start (Docker — Recommended) **The easiest and most reliable way** to get started on any machine: ```bash
cd docker
./setup.sh
``` This will:
1. Check Docker prerequisites
2. Create `.env` from `.env.example`
3. Build the Docker image for your platform (auto-detected)
4. Start PostgreSQL and dev environment
5. You're ready to build! Then enter the container and build:
```bash
docker exec -it sequoia-dev bash
cmake --preset debug && cmake --build build/debug
``` ### Alternative: VS Code Dev Container Open this project in VS Code and:
1. Install the "Dev Containers" extension
2. Press `F1` → "Dev Containers: Reopen in Container"
3. Wait for build (first time takes ~10 min) Everything is configured automatically! See [docker/README.md](docker/README.md) for detailed Docker documentation. --- ## Architecture Overview ```
HAL Client (embedded) ──gRPC (port 9090)──► HalServiceImpl │
Web UI/ REST clients ──HTTP (port 8080)──► Drogon REST API │ ┌──────────┴──────────┐ │ Service Layer │ │ RunService │ │ LogService │ │ AuthService │ │ OtaService ... │ └──────────┬──────────┘ │ ┌──────────┴──────────┐ │ Repository Layer │ │ (Drogon ORM/ PG) │ └──────────┬──────────┘ │ PostgreSQL 15+
``` --- ## Prerequisites (Native Build) **Note:** Using Docker (see Quick Start above) avoids all manual dependency installation. For native builds without Docker: | Tool | Version |
|------|---------|
| CMake | 3.22+ |
| vcpkg | latest |
| C++ compiler | GCC 12+ or Clang 16+ |
| PostgreSQL | 15+ |
| protoc + grpc_cpp_plugin | matching vcpkg grpc version | Set `VCPKG_ROOT` in your environment before building. --- ## Quick Start (Local Dev) ```bash
cp .env.example .env
# Edit .env — at minimum set SEQUOIA_DB_* and SEQUOIA_SESSION_SECRET # Start Postgres
docker compose -f docker/docker-compose.yml up postgres -d # Build and run (loads .env automatically)
./scripts/run_dev.sh
``` Or with Docker Compose (full stack): ```bash
docker compose -f docker/docker-compose.yml up --build
``` --- ## Building ```bash
# Debug (default dev preset)
cmake --preset debug
cmake --build build/debug # Release
cmake --preset release
cmake --build build/release # Run tests
cmake --preset test
cmake --build build/test
ctest --test-dir build/test --output-on-failure # ASan
cmake --preset asan
cmake --build build/asan
``` --- ## Configuration All configuration is via environment variables. See `.env.example` for the full list. | Variable | Default | Description |
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
| `SEQUOIA_LOG_LEVEL` | `info` | Log level (trace/debug/info/warn/error) | --- ## Module Registry Modules are declared in a YAML file (path set by `SEQUOIA_REGISTRY_PATH`): ```yaml
modules: - id: "com.example.temperature" name: "Temperature Monitor" description: "Reads temperature from sensor A" version: "1.2.0" result_type_url: "type.googleapis.com/sequoia.TempResult" config_type_url: "type.googleapis.com/sequoia.TempConfig" modes: ["single_shot", "monitoring"]
``` --- ## Database Migrations SQL migration files live in `migrations/`. They are applied automatically at startup in alphabetical order. Migrations are idempotent (tracked in `schema_migrations`). To add a new migration, create `migrations/NNNN_description.sql`. --- ## API All protected endpoints require either:
- A session cookie (from `POST/api/v1/auth/login`), or
- An `Authorization: Bearer <token>` header. ### Key endpoints | Method | Path | Description |
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
| POST | `/api/v1/ota/{id}/trigger` | Trigger OTA upgrade | --- ## Decisions Log | Decision | Rationale |
|----------|-----------|
| gRPC sync API for unary, streaming wait-loop | Simpler than async CQ for initial impl; async CQ adds complexity without benefit at this scale |
| Drogon for REST | Built-in PG ORM, async, WebSocket, SSE, session management |
| argon2id for passwords | OWASP recommended |
| SHA-256 for API tokens | Tokens are long random strings; SHA-256 is fast and sufficient |
| 1-minute minimum cron resolution | Full cron parser adds a dependency; document the limitation |
| Single `board` row (unique partial index) | The system targets exactly one physical test board |
| Migrations at startup | Ensures schema is always up to date; no separate migration tool needed |
| vcpkg manifest mode | Reproducible dependency pinning per repo | ## Build ```sh
cmake -B build -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build build -j$(nproc)
``` Binary: `build/sequoia-server` ## Run (dev) ```sh
./build/sequoia-server --config etc/sequoia/server.toml --log-level debug
``` Without a config file the binary starts with compiled-in defaults. ## Test ```sh
cmake --build build --target config-test
ctest --test-dir build --output-on-failure
``` ## Config Default path: `/etc/sequoia/server.toml`. See `etc/sequoia/server.toml` for all keys and defaults. CLI flags override file values: | Flag | Default |
|------|---------|
| `--config PATH` | `/etc/sequoia/server.toml` |
| `--log-level LEVEL` | `info` | ## Implementation Status | Step | Status |
|------|--------|
| 1 · Skeleton + config + logger | ✓ done |
| 2 · DB layer + migrations | — |
| 3 · DB CRUD (runs + batches) | — |
| 4 · Ingest queue + result writer | — |
| 5 · gRPC bidi server | — |
| … | — |
