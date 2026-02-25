# RAGFlow Microservices — Monorepo Structure

Target directory layout for the **new, dedicated monorepo** containing backend microservices and Angular frontend only. **No existing RAGFlow monolith code lives in this repo** — it is a standalone restructuring. **Planning only; confirm before implementation.**

---

## Table of Contents

1. [Monorepo Layout](#1-monorepo-layout)
2. [Backend Services](#2-backend-services)
3. [Frontend (Angular)](#3-frontend-angular)
4. [Shared and Infrastructure](#4-shared-and-infrastructure)
5. [Conventions](#5-conventions)

---

## 1. Monorepo Layout

The structure below is the **entire** contents of the new repo. There is no `api/`, `rag/`, `deepdoc/`, `agent/`, or `web/` from the original RAGFlow codebase.

```
<new-repo-name>/
├── backend/                    # All microservices (new implementations)
│   ├── identity-service/       # Auth, user, tenant (e.g. Node.js or Go)
│   ├── knowledge-base-service/ # KB/dataset CRUD (e.g. Node.js or Java)
│   ├── document-service/      # Document lifecycle, chunk metadata (e.g. Java or Go)
│   ├── parsing-service/       # deepdoc: PDF, OCR, parsers (e.g. Python or Go)
│   ├── rag-service/           # Chunking, embedding, retrieval, rerank (e.g. Python or Go)
│   ├── chat-service/          # Conversations, dialogs (e.g. Node.js or Java)
│   ├── search-service/        # Product search API (e.g. Node.js or Java)
│   ├── agent-service/         # Canvas, agents, tools (e.g. Python or Node.js)
│   ├── file-service/         # File upload/download, MinIO (e.g. Go or Node.js)
│   ├── llm-gateway-service/   # LLM config, proxy (e.g. Node.js or Go)
│   ├── connector-service/     # Plugins, data sources (e.g. Node.js)
│   └── admin-service/         # System settings, admin API (e.g. Node.js or Java)
├── frontend/                   # Single Angular application
│   ├── src/
│   ├── angular.json
│   ├── package.json
│   └── ...
├── packages/                   # Optional: shared API contracts, types, SDKs
│   ├── api-contracts/         # OpenAPI specs, shared DTOs
│   └── sdk-js/                # Optional JS/TS client for frontend
├── gateway/                    # API gateway (Nginx configs or custom gateway app)
│   ├── nginx/
│   └── ...
├── docker/                     # All Docker for this repo (no monolith)
│   ├── docker-compose-base.yml # Infra only: MySQL, Redis, MinIO, ES/Infinity (copy/adapt from RAGFlow)
│   └── docker-compose.microservices.yml  # Gateway + all microservice containers
├── docs/                       # Optional: architecture, runbooks (this planning can live here or in repo root)
│   ├── INDEX.md
│   ├── HIGH_LEVEL_OVERVIEW.md
│   ├── STEP_BY_STEP_PLAN.md
│   ├── MONOREPO_STRUCTURE.md
│   └── DOCKER_STRATEGY.md
├── .env.example
├── README.md
└── .gitignore
```

*Note: This planning documentation currently lives in the original RAGFlow repo under `microservices/`. When you create the new repo, copy it into `docs/` (or keep at root) so the new repo is self-contained.*

## 2. Backend Services

Each service lives under `backend/<service-name>/` and can use a different language/framework.

### 2.1 Per-Service Directory Convention

Example for a **Node.js** service:

```
backend/identity-service/
├── src/
│   ├── index.ts
│   ├── routes/
│   ├── services/
│   └── ...
├── package.json
├── tsconfig.json
├── Dockerfile
└── README.md
```

Example for a **Java** service (e.g. Spring Boot):

```
backend/document-service/
├── src/
│   └── main/
│       ├── java/
│       └── resources/
├── pom.xml
├── Dockerfile
└── README.md
```

Example for **Go**:

```
backend/file-service/
├── cmd/
├── internal/
├── go.mod
├── go.sum
├── Dockerfile
└── README.md
```

Example for **Python** (e.g. parsing-service or rag-service):

```
backend/parsing-service/
├── src/
├── requirements.txt
├── Dockerfile
└── README.md
```

### 2.2 Service Responsibilities (Quick Reference)

| Service | Suggested stack | Main artifact / port |
|---------|-----------------|----------------------|
| identity-service | Node.js or Go | HTTP API (e.g. 9401) |
| knowledge-base-service | Node.js or Java | HTTP API (e.g. 9402) |
| document-service | Java or Go | HTTP API (e.g. 9403) |
| parsing-service | Python or Go | HTTP API (e.g. 9404) |
| rag-service | Python or Go | HTTP API (e.g. 9405) |
| chat-service | Node.js or Java | HTTP API (e.g. 9406) |
| search-service | Node.js or Java | HTTP API (e.g. 9407) |
| agent-service | Python or Node.js | HTTP API (e.g. 9408) |
| file-service | Go or Node.js | HTTP API (e.g. 9409) |
| llm-gateway-service | Node.js or Go | HTTP API (e.g. 9410) |
| connector-service | Node.js | HTTP API (e.g. 9411) |
| admin-service | Node.js or Java | HTTP API (e.g. 9412) |

Port numbers are placeholders; final ports in Docker Compose and gateway config.

---

## 3. Frontend (Angular)

- **Single app** under `frontend/`.
- Consumes only the **API gateway** (same origin or configured base URL for `/v1`, `/api/v1`).
- No direct calls to individual microservices from the browser (except optional server-side proxy in gateway).

Suggested layout:

```
frontend/
├── src/
│   ├── app/
│   │   ├── core/           # Auth, guards, interceptors
│   │   ├── features/       # Lazy-loaded modules (datasets, chat, agent, etc.)
│   │   ├── shared/         # Shared components, pipes, services
│   │   └── app.config.ts
│   ├── index.html
│   └── main.ts
├── angular.json
├── package.json
└── README.md
```

- **Build**: `npm run build` (or `ng build`) produces static assets; served by gateway or a separate web server (e.g. Nginx) that proxies API to gateway.

---

## 4. Shared and Infrastructure

- **packages/api-contracts**: OpenAPI 3 specs per domain (or one combined); optional codegen for server/client.
- **packages/sdk-js**: Optional TypeScript/JavaScript client used by Angular to call gateway; can be generated from OpenAPI or hand-written.
- **gateway**: Nginx configs (e.g. `gateway/nginx/conf.d/`) or a small gateway application (e.g. Node/Go) that routes by path to backend services and validates JWT.

---

## 5. Conventions

- **Naming**: Kebab-case for directories and service names (`identity-service`, not `identityService`).
- **API prefix**: Services behind gateway expose same paths as today (e.g. `/v1/knowledgebase`, `/v1/document`); gateway strips prefix or routes by path.
- **Config**: Each service reads config from env (and optionally a config file); no shared config repo required initially; Docker Compose or K8s inject env.
- **Logs**: JSON logs to stdout; same as current RAGFlow practice for Docker.
- **Health**: Each service exposes `/health` or `/healthz` for readiness/liveness.
- **Database**: **One database per service**; each service connects only to its own DB. See [DATABASE_PER_SERVICE.md](./DATABASE_PER_SERVICE.md) for the full table → service mapping and deployment options.

---

## Document Links

- [Index](./INDEX.md)
- [High-level overview](./HIGH_LEVEL_OVERVIEW.md)
- [Step-by-step plan](./STEP_BY_STEP_PLAN.md)
- [Database per service](./DATABASE_PER_SERVICE.md)
- [Docker strategy](./DOCKER_STRATEGY.md)
