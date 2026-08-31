# Sequoia-Server — Docker image build/run helpers.
# The top-level Makefile passes IMAGE_NAME=$(DOCKER_SERVER_IMAGE) so the image
# Docker then runs matches the tag produced by build-server.
IMAGE_NAME ?= sequoia

.PHONY: help check setup build build-amd64 build-arm64 build-server up down logs shell clean run delete-images

help:
	@echo "Sequoia Docker Makefile"
	@echo ""
	@echo "Recommended commands:"
	@echo " make build-server - Build the x86_64 server image (production)"
	@echo " make build-arm64  - Build for ARM64 target (Intel x86 RT 5)"
	@echo " make up / down / shell / logs  - dev environment (docker-compose)"

setup:
	@./setup.sh

check:
	@./check-prereqs.sh

build:
	@./build.sh

build-amd64:
	@docker buildx build --platform linux/amd64 --load \
		--build-arg UID=$$(id -u) --build-arg GID=$$(id -g) \
		-f Dockerfile.unified -t $(IMAGE_NAME):amd64 ..

build-arm64:
	@docker buildx build --platform linux/arm64 --load \
		--build-arg UID=$$(id -u) --build-arg GID=$$(id -g) \
		-f Dockerfile.unified -t $(IMAGE_NAME):arm64 ..

# The top-level Makefile passes IMAGE_NAME=$(DOCKER_SERVER_IMAGE) so the image
# Docker then runs matches the tag produced here.
build-server:
	@echo "Building Docker image $(IMAGE_NAME) for x86 Ubuntu 22.04..."
	docker buildx build --platform linux/amd64 --load \
		--build-arg UID=$$(id -u) --build-arg GID=$$(id -g) \
		-f Dockerfile.server -t $(IMAGE_NAME) ..

up:
	@docker compose -f docker-compose.dev.yml up -d
	@echo "Services started. Enter container with: make shell"

down:
	@docker compose -f docker-compose.dev.yml down

logs:
	@docker compose -f docker-compose.dev.yml logs -f

shell:
	@docker exec -it sequoia-dev bash || echo "Container not running. Start with: make up"

run:
	docker run -it --rm --hostname "Intel x86 RT 5" $(IMAGE_NAME)

clean:
	docker rmi $(IMAGE_NAME) || true

delete-images:
	@echo "Warning: This will delete ALL Docker images!"
	@read -p "Are you sure? (y/N) " -n 1 -r; \
	echo; \
	if [[ $$REPLY =~ ^[Yy]$$ ]]; then \
		docker rmi $$(docker images -q) || true; \
	fi
