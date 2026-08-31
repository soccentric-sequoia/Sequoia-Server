# Sequoia-Server Build and Deployment

## Scope

Sequoia-Server is the C++ server component built with CMake/Ninja. Its root
Makefile provides native output in `build/native`, an ARM64 Docker build for
the device-side path, a separate x86 Ubuntu Docker build for the server path,
CTest, CPack, remote deployment, and documentation/quality targets. Docker
compose and helper scripts provide local database and development setup.

## 1. Prerequisites

- Native: CMake, Ninja, compiler, Protobuf/gRPC tooling, PostgreSQL client
  libraries, OpenSSL, and the dependencies baked into the Docker build image.
- Docker path: Docker/buildx and permission to run containers. `docker/build.sh`
  and `docker/check-prereqs.sh` support the image workflow.
- All toolchain dependencies come from the Docker image (`docker/Makefile build-server`); no host setup script is needed.
  that option when sudo/apt and network access are available. It can build
  yaml-cpp, Drogon, and `grpc_cpp_plugin` from source.
- Remote deployment: `sshpass`, reachable targets, and credentials. **Credential
  and device required.**

## 2. Setup

## 3. Build

```sh
make configure                  # native CMake/Ninja, tests enabled
make build                      # native build
make docker-image               # conditional ARM64 image sequoia
make docker-build               # ARM64 cross build
make docker-image-server        # conditional x86 image sequoia-dev
make docker-build-server        # x86 Ubuntu server build
```

The repository also documents `cmake --preset debug` and
`cmake --build build/debug` for the Docker development environment; use the
root Makefile targets for the native/cross layout described above.

## 4. Test and lint

```sh
make test                       # native CTest
make docker-test                # ARM64/QEMU CTest
make docker-test-server        # x86 server CTest
make format
make cppcheck
make doc                         # Doxygen via ninja doc_doxygen
```

## 5. Artifacts

The root Makefile has no CPack package target. Native and Docker build artifacts
are under their respective build directories, while `docker/Dockerfile*` and
`docker-compose*.yml` define images and local services. For local database-backed
development, use `docker/docker-compose.dev.yml` and the migrations under
`migrations/`.

## 6. Deploy, install, and run

```sh
make run [APP] [ARGS...]         # local test applications
make deploy-a IP_A=... USER=... PASS='...'
make deploy-b IP_B=... USER=... PASS='...'
make run-a [APP] [ARGS...]
make run-b [APP] [ARGS...]
make list
```

Deployment copies the cross-built `lib/` and `bin/` trees into `/tmp/acu-test`
on the target. For the standalone server development environment, see
`docker/docker-compose.dev.yml` and `docker/build.sh`; the
compose stack is local infrastructure, not cloud provisioning.

## 7. Cleanup

```sh
make clean                       # remove native/cross build directories
make docker-clean               # clean ARM64 Docker build directory
make docker-clean-server        # clean x86 server Docker build directory
docker compose -f docker/docker-compose.dev.yml down
```

## 8. Verification caveats

ARM64 emulation and x86 containers do not validate a deployed server's network,
database, TLS, gRPC, or device integration behavior. Remote targets and
credentials are conditional; inspect logs, migrations, service health, and
protocol behavior manually. Never reuse the development `.env` credentials or
compose defaults for a cloud/production deployment.