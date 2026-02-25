# RAGFlow Microservices — Docker Strategy

How Docker is used in the **new, dedicated monorepo**. The new repo contains its own `docker/` with a base compose (copied/adapted from RAGFlow) and a microservices compose. **No monolith code or container lives in this repo.** **Planning only; confirm before implementation.**

---

## Table of Contents

1. [Compose Files](#1-compose-files)
2. [Base Compose (Existing)](#2-base-compose-existing)
3. [Microservices Compose (New)](#3-microservices-compose-new)
4. [Gateway and Routing](#4-gateway-and-routing)
5. [Running Modes](#5-running-modes)

---

## 1. Compose Files

All compose files live in the **new repo** under `docker/`. The original RAGFlow repo is not required at runtime.

| File | Purpose |
|------|---------|
| `docker/docker-compose-base.yml` | **In the new repo.** Infra only: MySQL, Redis, MinIO, Elasticsearch/Infinity (by profile), etc. Copy or adapt from RAGFlow’s `docker-compose-base.yml`; no application containers. |
| `docker/docker-compose.microservices.yml` | **In the new repo.** Includes base; defines API gateway + all microservice containers. **No monolith service** — this repo only runs microservices. |

Relationship:

- **Base** = infrastructure only (same as RAGFlow’s base; brought into the new repo).
- **Microservices compose** = base + gateway + identity-service + … + admin-service. All services are implemented in this repo.
- **Original RAGFlow repo** = separate; if you need to run the old monolith in parallel (e.g. strangler), run it from that repo and point the gateway at its URL; the new repo itself never contains or runs the monolith.

---

## 2. Base Compose (In the New Repo)

- **Path**: `docker/docker-compose-base.yml` **in the new monorepo**.
- **Source**: Copy or adapt from the original RAGFlow repository’s `docker-compose-base.yml`. It provides:
  - MySQL
  - Redis
  - MinIO
  - Elasticsearch (profile: elasticsearch) / Infinity (profile: infinity) / etc.
  - Optional: Kibana, TEI, sandbox-executor-manager, etc.
- All microservices connect to these via the same network and env (e.g. `MYSQL_HOST`, `REDIS_HOST`, `MINIO_*`). No monolith container is defined in this repo.

---

## 3. Microservices Compose

- **Path**: `docker/docker-compose.microservices.yml` **in the new repo**.
- **Include** base so that infra is started:

  ```yaml
  include:
    - ./docker-compose-base.yml
  ```

- **Define**:
  - **api-gateway**: Single entrypoint (e.g. Nginx or custom gateway). Ports 80/443 and/or 9380 for API. Proxies to backend services by path.
  - **identity-service**, **knowledge-base-service**, **document-service**, **parsing-service**, **rag-service**, **chat-service**, **search-service**, **agent-service**, **file-service**, **llm-gateway-service**, **connector-service**, **admin-service**: One service per container; each with its own Dockerfile under `backend/<service-name>/`.
  - **No monolith container** — all logic lives in these services. If you run the old RAGFlow app during transition, it runs from the **other** repository and is reached by URL (e.g. gateway routes to that URL for paths not yet implemented).

- **Network**: All services on the same network (e.g. `ragflow`) so they resolve by service name (e.g. `http://identity-service:9401`).
- **Env**: Use `.env` in the new repo; add service-specific vars as needed. Only gateway (and optionally debug ports) need to be exposed on the host.

Example skeleton (not implemented):

```yaml
# docker/docker-compose.microservices.yml
include:
  - ./docker-compose-base.yml

services:
  api-gateway:
    build: ../gateway   # or image
    ports:
      - "${SVR_WEB_HTTP_PORT:-80}:80"
      - "${SVR_HTTP_PORT:-9380}:9380"
    env_file: .env
    depends_on:
      - identity-service
    networks:
      - ragflow

  identity-service:
    build: ../backend/identity-service
    env_file: .env
    depends_on:
      mysql: { condition: service_healthy }
      redis: { condition: service_healthy }
    networks:
      - ragflow

  knowledge-base-service:
    build: ../backend/knowledge-base-service
    # ...
  # ... other services ...
```

---

## 4. Gateway and Routing

- **Role**: Single entrypoint for browser and API clients. Reverse-proxy by path to the correct backend service.
- **Options**:
  1. **Nginx**: Config (e.g. under `gateway/nginx/`) with `location` blocks that `proxy_pass` to `http://identity-service:9401`, etc.
  2. **Custom gateway**: Small Node/Go app that reads a routing table and proxies; can add JWT validation and headers (e.g. `X-User-Id`, `X-Tenant-Id`).
- **Routing table**: As in [HIGH_LEVEL_OVERVIEW.md](./HIGH_LEVEL_OVERVIEW.md#5-api-gateway--routing). All paths go to microservices in this repo. If some features are not yet implemented, the gateway can optionally route those paths to an **externally running** monolith (different repo/deployment) by URL; the monolith is never part of this repo.

---

## 5. Running Modes

- **Development (microservices only)**  
  1. From the **new repo**: `docker compose -f docker/docker-compose.microservices.yml up -d`  
  This starts base infra + gateway + all microservices. Frontend can be run locally (Angular dev server) with proxy to gateway, or served by the gateway if the frontend is built into the gateway image.

- **Optional: Strangler (monolith elsewhere)**  
  If the old RAGFlow monolith is still running (from the **original repo**), you can point the gateway at its URL for paths not yet implemented in the new services. The monolith is not in this repo and is not started by this repo’s compose.

- **Production**  
  Same compose files in the new repo; use production builds and secrets. Start base + microservices from this repo only.

---

## Document Links

- [Index](./INDEX.md)
- [High-level overview](./HIGH_LEVEL_OVERVIEW.md)
- [Step-by-step plan](./STEP_BY_STEP_PLAN.md)
- [Monorepo structure](./MONOREPO_STRUCTURE.md)
- [Database per service](./DATABASE_PER_SERVICE.md)
