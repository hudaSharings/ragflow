# RAGFlow Microservices — High-Level Overview

High-level strategy for decomposing the RAGFlow monolith into microservices. **Planning only; no implementation until you confirm.**

---

## Table of Contents

1. [Objectives and Principles](#1-objectives-and-principles)
2. [Decomposition Strategy](#2-decomposition-strategy)
3. [Proposed Microservices](#3-proposed-microservices)
4. [Service Boundaries and Responsibilities](#4-service-boundaries-and-responsibilities)
5. [API Gateway & Routing](#5-api-gateway--routing)
6. [Data and Integration Considerations](#6-data-and-integration-considerations)
7. [Monorepo and Technology Choices](#7-monorepo-and-technology-choices)

---

## 1. Objectives and Principles

### 1.1 Goals

- **Decompose** the single RAGFlow application into independently deployable microservices.
- **New, dedicated monorepo**: One repository containing **only** backend microservices and a single frontend (Angular). **No existing RAGFlow monolith code** — the new repo is a full restructuring; the current RAGFlow codebase stays in a separate repo and is used only as a reference for behavior and APIs.
- **Infrastructure**: Copy or adapt the RAGFlow base Docker Compose (MySQL, Redis, MinIO, ES/Infinity, etc.) into the new repo’s `docker/`; add a microservices compose to run gateway + all services. No monolith container in the new repo.
- **Polyglot backends**: Each service may be implemented in Node.js, Java, or Go (or Python where it fits) as appropriate.

### 1.2 Principles

- **New repo, no monolith code**: All deliverables live in the new monorepo; logic is ported or reimplemented from the RAGFlow reference, not copied.
- **Domain-driven boundaries**: Services align to business capabilities (auth, knowledge base, documents, chat, agents, etc.).
- **Shared nothing where possible**: Prefer dedicated data per service; share only via APIs or events when necessary.
- **API-first**: Each service exposes a clear REST (or gRPC) API; frontend and gateway call services only.

---

## 2. Decomposition Strategy

### 2.1 Current Monolith Mapping (Reference — In a Separate Repo)

When porting behavior into the new services, use the **original RAGFlow repo** as reference. This table maps that codebase to service boundaries; none of this code lives in the new monorepo.

| Monolith Area | Location | Role |
|---------------|----------|------|
| API / Auth | `api/apps/__init__.py`, `user_app`, `tenant_app`, `auth/` | Routing, JWT/session auth, tenant context |
| Knowledge Base | `kb_app`, `knowledgebase_service` | Dataset/KB CRUD, configuration |
| Documents | `document_app`, `file_app`, `file2document_app`, `chunk_app` | Upload, parsing, chunking, storage |
| RAG Core | `rag/` (nlp, llm, flow, utils) | Search, embedding, rerank, chunking pipelines |
| Document Processing | `deepdoc/` | Parsers, OCR, layout |
| Chat / Conversations | `dialog_app`, `conversation_app` | Dialogs, chat sessions, messages |
| Search (product) | `search_app` | Product-level search UX/API |
| Agents / Canvas | `canvas_app`, `agent/` | Workflow canvas, agent runs, tools |
| LLM / Model Config | `llm_app` | LLM model configuration |
| Files | `file_app` | File metadata and storage abstraction |
| Plugins / Connectors | `plugin_app`, `connector_app` | Extensions, data source connectors |
| SDK / External API | `api/apps/sdk/*` | External developer API |
| Admin | `system_app`, admin routes | System settings, admin operations |
| MCP / Eval / Langfuse | `mcp_server_app`, `evaluation_app`, `langfuse_app` | MCP server, eval, observability |

### 2.2 Decomposition Approach

- **Extract by bounded context**: Each service in the new repo implements one or more of these areas; logic is **ported or reimplemented** from the reference repo.
- **Identify shared kernels**: Auth/identity and tenant context become the **Identity** service; other services depend on it (JWT or user/tenant headers).
- **Heavy processing as separate services**: Document parsing and RAG indexing/retrieval are separate services for scalability and language choice.
- **Frontend**: Single Angular app in the new monorepo; it calls the API gateway, which routes to microservices only (no monolith in this repo).

---

## 3. Proposed Microservices

Suggested service set. Names and boundaries are for you to confirm.

| # | Service Name | Suggested Tech | Primary Responsibility |
|---|--------------|----------------|------------------------|
| 1 | **identity-service** | Node.js or Go | Auth (login, JWT, API tokens), tenant and user context |
| 2 | **knowledge-base-service** | Node.js or Java | KB/dataset CRUD, configuration, membership |
| 3 | **document-service** | Java or Go | Document lifecycle, file–document linking, chunk metadata, orchestration of parsing |
| 4 | **parsing-service** | Python or Go | deepdoc: PDF/OCR/layout, format-specific parsers |
| 5 | **rag-service** | Python or Go | RAG pipeline: chunking, embedding, indexing into ES/Infinity, retrieval, rerank |
| 6 | **chat-service** | Node.js or Java | Conversations, dialogs, chat messages, session state |
| 7 | **search-service** | Node.js or Java | Product-level search API consumed by frontend |
| 8 | **agent-service** | Python or Node.js | Agent workflows (canvas), execution, tools (Tavily, SQL, etc.) |
| 9 | **file-service** | Go or Node.js | File upload/download, MinIO/S3 abstraction, metadata |
| 10 | **llm-gateway-service** | Node.js or Go | LLM model config, routing to backends (optional aggregation) |
| 11 | **connector-service** | Node.js | Plugins, data source connectors |
| 12 | **admin-service** | Node.js or Java | System settings, admin-only APIs, monitoring hooks |

Optional / later:

- **sdk-gateway**: External developer API (can be part of API gateway or a dedicated service).
- **mcp-service**: MCP server functionality (can stay in monolith initially or split later).
- **evaluation-service** / **observability**: Eval and Langfuse integration (can be a small service or part of admin/rag).

---

## 4. Service Boundaries and Responsibilities

### 4.1 Identity Service

- Login, logout, JWT issue/validate, API token management.
- Tenant and user resolution; expose “current user/tenant” to other services via validated tokens or shared headers.
- **Data**: Users, tenants, API tokens (or delegate to existing MySQL schema with clear ownership).

### 4.2 Knowledge Base Service

- Create/update/delete knowledge bases (datasets).
- KB-level configuration (chunking, retrieval settings).
- **Data**: Knowledge base metadata; references to documents (IDs) and chunks (managed by document/rag services).

### 4.3 Document Service

- Create/update/delete document metadata; link files to documents and to a KB.
- Trigger parsing (call parsing-service) and RAG indexing (call rag-service).
- Track document and chunk job status.
- **Data**: Document and chunk metadata (and possibly job queue); actual blob storage via file-service.

### 4.4 Parsing Service

- Input: file reference (e.g. from MinIO) or stream.
- Output: parsed structure (text, tables, layout); optionally chunk boundaries.
- Uses current deepdoc logic (parsers, OCR, vision).
- **Data**: Stateless or minimal; reads from storage, returns structured result.

### 4.5 RAG Service

- Chunking strategies, embedding, indexing into Elasticsearch/Infinity.
- Retrieval, rerank, optional advanced RAG (e.g. graph).
- **Data**: Vector index (ES/Infinity); may own chunk text and vectors or rely on document-service for metadata.

### 4.6 Chat Service

- Create/list conversations; create/list messages; dialog configuration.
- May call rag-service for retrieval and llm-gateway for completion.
- **Data**: Conversations, dialogs, messages.

### 4.7 Search Service

- Product-level search (e.g. “search across my datasets/chats”).
- Composes rag-service and possibly chat-service; exposes a simple API for the frontend.

### 4.8 Agent Service

- Agent/workflow definitions (canvas), runs, tool execution (Tavily, SQL, etc.).
- **Data**: Workflow definitions, run history; may use same DB as chat or separate.

### 4.9 File Service

- Upload to MinIO/S3, download URLs or stream.
- File metadata (name, size, tenant, owner).
- **Data**: File metadata; blob storage in MinIO.

### 4.10 LLM Gateway Service

- Manage LLM model configuration (per tenant/user).
- Route completion/embedding requests to appropriate backends (OpenAI, Ollama, etc.); can be a thin proxy or aggregation layer.

### 4.11 Connector Service

- Plugin and data source connector management; sync jobs (e.g. sync data source into KB).
- **Data**: Connector configs, sync state.

### 4.12 Admin Service

- System-wide settings, health, admin user management (if not in identity-service), monitoring endpoints.
- **Data**: System config, admin audit log.

---

## 5. API Gateway & Routing

- **Single entrypoint** for the frontend and external API (e.g. `/api/v1/*`, `/v1/*`).
- **Gateway** responsibilities:
  - Route by path to the correct microservice (all running from the new repo).
  - Optional: JWT validation (or delegate to identity-service); add user/tenant headers for downstream services.
  - CORS, rate limiting, request logging.
- **Options**: Nginx, Kong, APISIX, or a small custom gateway (e.g. Node/Go).
- **Routing table** (conceptual):
  - `/v1/user*`, `/v1/tenant*`, login → identity-service.
  - `/v1/knowledgebase*`, `/v1/dataset*` → knowledge-base-service.
  - `/v1/document*`, `/v1/chunk*`, `/v1/file2document*` → document-service.
  - `/v1/file*` → file-service.
  - `/v1/dialog*`, `/v1/conversation*` → chat-service.
  - `/v1/search*` (product search) → search-service.
  - `/v1/canvas*`, `/v1/agent*` → agent-service.
  - `/v1/llm*` → llm-gateway-service.
  - `/v1/connector*`, `/v1/plugin*` → connector-service.
  - `/v1/admin*`, `/api/v1/admin*` → admin-service.
  - `/v1/api*` (API tokens, etc.) → identity-service.
  - Any remaining paths → 404 or the appropriate service once implemented.

---

## 6. Data and Integration Considerations

### 6.1 Database per Service

- **Each microservice has its own database** (or dedicated schema). No service connects to another service’s database. See [DATABASE_PER_SERVICE.md](./DATABASE_PER_SERVICE.md) for:
  - Mapping of every current RAGFlow table to a service
  - Per-service database summary (which tables live where)
  - Cross-service references (by ID only, no shared FKs)
  - Deployment options (one MySQL with multiple databases vs multiple instances)
- **Shared infrastructure** (conceptually):
  - **MySQL**: One instance with multiple databases (e.g. `identity_db`, `document_db`, …), or one DB instance per service. Each service connects only to its own DB.
  - **Redis**: Sessions, caches, task queues; use separate key prefixes or DB numbers per service where possible.
  - **MinIO**: Shared object storage; access via file-service or with tenant/path isolation.
  - **Elasticsearch / Infinity**: Used by rag-service for vector/search index; other services do not access it directly.

### 6.2 Inter-Service Communication

- **Sync**: REST (JSON) for most CRUD and query flows.
- **Async** (optional): Events (e.g. “document.parsed”, “chunk.indexed”) for parsing and RAG pipeline; can use Redis Streams or a message broker later.
- **Identity**: Services accept JWT or a signed “user-id/tenant-id” header set by the gateway or identity-service.

### 6.3 Data Consistency

- Prefer **eventual consistency** across services; avoid distributed transactions.
- **Ownership**: Each service is the source of truth for its entities; others reference by ID only (see [DATABASE_PER_SERVICE.md](./DATABASE_PER_SERVICE.md#5-cross-service-references-and-no-shared-fks)).
- **Porting from RAGFlow**: When reimplementing, create per-service databases and migrate or seed data per the table → service mapping; no monolith in this repo to dual-write with.

---

## 7. Monorepo and Technology Choices

- **Monorepo** = **new, dedicated repository** containing:
  - **Backend**: One subdirectory per microservice under `backend/`. Each can be Node.js, Java, Go, or Python. **No monolith code** from RAGFlow.
  - **Frontend**: Single Angular app under `frontend/`.
  - **Shared**: API contracts (OpenAPI), shared types, or client SDKs in `packages/` or `shared/`.
  - **Gateway**: Nginx or custom app under `gateway/`.
  - **Docker**: `docker-compose-base.yml` (copied/adapted from RAGFlow) and `docker-compose.microservices.yml` in the new repo; no monolith container.
- The **original RAGFlow repository** remains separate and is used only as a reference for APIs, data models, and behavior when implementing the new services.

See [MONOREPO_STRUCTURE.md](./MONOREPO_STRUCTURE.md) for the target directory layout and [DOCKER_STRATEGY.md](./DOCKER_STRATEGY.md) for compose usage.

---

## Document Links

- [Index](./INDEX.md)
- [Step-by-step plan](./STEP_BY_STEP_PLAN.md)
- [Monorepo structure](./MONOREPO_STRUCTURE.md)
- [Database per service](./DATABASE_PER_SERVICE.md)
- [Docker strategy](./DOCKER_STRATEGY.md)
