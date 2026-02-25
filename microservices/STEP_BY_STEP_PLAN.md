# RAGFlow Microservices — Step-by-Step Plan

Phased plan to build the **new, dedicated monorepo** (microservices + Angular). The monorepo does **not** contain the existing RAGFlow monolith; logic is ported or reimplemented. **Review and confirm before any implementation.**

---

## Table of Contents

1. [Phase Overview](#1-phase-overview)
2. [Step-by-Step Plan](#2-step-by-step-plan)
3. [Checkpoints and Rollback](#3-checkpoints-and-rollback)
4. [Risks and Mitigations](#4-risks-and-mitigations)

---

## 1. Phase Overview

| Phase | Focus | Outcome |
|-------|--------|---------|
| **0** | New repo & prep | Create dedicated repo; monorepo layout, base compose (copy/adapt from RAGFlow), gateway skeleton, Angular shell |
| **1** | Identity & Gateway | Auth and tenant as a service; gateway routes auth to it |
| **2** | File & Document | File and document lifecycle services; parsing/RAG either new services or stubbed |
| **3** | RAG & Parsing | Parsing-service and rag-service; document-service orchestrates them |
| **4** | Chat & Search | Chat and product search as services |
| **5** | Agents & Rest | Agent, LLM gateway, connector, admin services; frontend fully on gateway |
| **6** | Hardening | Optional DB split, events, docs, monitoring; no monolith in this repo |

---

## 2. Step-by-Step Plan

### Phase 0 — New Repo & Preparation

- **0.1** Create the **new dedicated repository** (no RAGFlow monolith code):
  - `backend/` with subdirs per service (empty or stub).
  - `frontend/` for Angular app (scaffold).
  - `packages/` or `shared/` for API contracts/types if desired.
  - `gateway/` for Nginx or minimal gateway app.
  - `docker/`: copy or adapt `docker-compose-base.yml` from RAGFlow; add `docker-compose.microservices.yml` with gateway + placeholder services.
- **0.2** **Database per service**: Define per-service databases (e.g. one MySQL instance with DBs `identity_db`, `document_db`, `file_db`, …). Document in [DATABASE_PER_SERVICE.md](./DATABASE_PER_SERVICE.md); each service’s env points to its own DB only.
- **0.3** Document gateway routing (path → service) and env for each backend.
- **0.4** Angular: create project, configure proxy/dev server to hit gateway for `/v1` and `/api/v1`.
- **0.5** CI: build and test backend services and frontend independently (per-directory or workspace scripts).

**Checkpoint**: New repo exists with full layout; base + microservices compose start; gateway returns 502/503 for unimplemented routes; Angular loads and can call the gateway.

---

### Phase 1 — Identity & API Gateway

- **1.1** Implement **identity-service** (Node.js/Go) in the new repo:
  - Login, logout, JWT issue/validate, API token CRUD.
  - User and tenant resolution; health check.
- **1.2** Identity data: **identity-service’s own database** (e.g. `identity_db`) with tables `user`, `tenant`, `user_tenant`, `invitation_code`, `api_token` per [DATABASE_PER_SERVICE.md](./DATABASE_PER_SERVICE.md); no dependency on any monolith.
- **1.3** Gateway: route `/v1/user*`, `/v1/tenant*`, login, and `/v1/api*` (token management) to identity-service; other paths return 404 or stub until implemented.
- **1.4** Frontend: point login and auth to gateway; JWT sent to gateway and downstream services.
- **1.5** Optional: gateway validates JWT and adds `X-User-Id`, `X-Tenant-Id` for downstream services.

**Checkpoint**: Login and auth work through identity-service via gateway. No monolith in this repo.

---

### Phase 2 — File & Document Services

- **2.1** Implement **file-service** (Go/Node.js): upload to MinIO, download URL or stream, file metadata CRUD. Use same MinIO bucket/paths as RAGFlow for compatibility if needed.
- **2.2** Gateway: route `/v1/file*` to file-service.
- **2.3** Implement **document-service** (Java/Go/Node.js): document metadata CRUD; link file ↔ document ↔ KB. “Start parsing” and “Start indexing” trigger: call parsing-service and rag-service (or stub them until Phase 3).
- **2.4** Gateway: route `/v1/document*`, `/v1/chunk*`, `/v1/file2document*` to document-service.
- **2.5** document-service uses **document_db** (tables: document, file2document, task, pipeline_operation_log); file-service uses **file_db** (table: file). Clear ownership; no monolith code in this repo.
- **2.6** Frontend: verify dataset document list and upload through gateway.

**Checkpoint**: File upload and document metadata go through new services. Parsing/indexing use new or stubbed services (no monolith).

---

### Phase 3 — RAG & Parsing Services

- **3.1** Implement **parsing-service** (Python or Go): input = file reference (e.g. MinIO path), output = parsed structure (and optionally chunk boundaries). Port behavior from RAGFlow’s deepdoc; stateless.
- **3.2** Implement **rag-service** (Python or Go): chunking, embedding, indexing to ES/Infinity; retrieval and rerank. Port from RAGFlow’s rag/; document-service or a task queue triggers “index this document” and calls rag-service.
- **3.3** document-service: call parsing-service then rag-service (sync or async via queue); no monolith.
- **3.4** Gateway: add internal or public routes for parsing/RAG if needed; otherwise document-service calls them by service name.
- **3.5** Task executor (Redis or queue consumer) lives in document-service or rag-service in this repo.

**Checkpoint**: Document parsing and RAG indexing run in new services; document-service orchestrates. All in the new repo.

---

### Phase 4 — Chat & Search Services

- **4.1** Implement **chat-service** (Node.js/Java): conversations, messages, dialog configuration; port data model from RAGFlow’s dialog_app + conversation_app.
- **4.2** Gateway: route `/v1/dialog*`, `/v1/conversation*` to chat-service.
- **4.3** Chat-service calls rag-service for retrieval and llm-gateway-service (Phase 5) for completion.
- **4.4** Implement **search-service** (Node.js/Java): product-level search API; calls rag-service (and optionally chat-service).
- **4.5** Gateway: route `/v1/search*` to search-service.
- **4.6** Frontend: verify chat and search UX through gateway.

**Checkpoint**: Chat and product search served by new services. No monolith in this repo.

---

### Phase 5 — Agents, LLM, Connectors, Admin

- **5.1** Implement **agent-service** (Python/Node.js): canvas/workflow definitions, runs, tool execution (Tavily, SQL, etc.); port from RAGFlow’s canvas_app and agent/.
- **5.2** Gateway: route `/v1/canvas*`, `/v1/agent*` to agent-service.
- **5.3** Implement **llm-gateway-service** (Node.js/Go): model config, proxy to OpenAI/Ollama/etc.; chat-service and agent-service call it.
- **5.4** Gateway: route `/v1/llm*` to llm-gateway-service.
- **5.5** Implement **connector-service** (Node.js): plugins, data source connectors; route `/v1/connector*`, `/v1/plugin*`.
- **5.6** Implement **admin-service** (Node.js/Java): system settings, admin APIs; route `/v1/admin*`, `/api/v1/admin*`.
- **5.7** Implement remaining SDK or special routes (e.g. MCP, evaluation) in the appropriate services.
- **5.8** Frontend: all features use gateway and microservices only.

**Checkpoint**: All core flows implemented in the new repo; no monolith dependency.

---

### Phase 6 — Hardening & Optional Enhancements

- **6.1** This repo never contained the monolith; no “retire monolith” step here. Ensure gateway routes all paths to the correct microservices; any unimplemented path returns 404 or is implemented.
- **6.2** **Database per service** is the default: each service already has its own DB (or schema). Optionally move from “one MySQL, multiple databases” to separate MySQL instances per service if you need stronger isolation or independent scaling. See [DATABASE_PER_SERVICE.md](./DATABASE_PER_SERVICE.md).
- **6.3** (Optional) Message queue or events for document.parsed / chunk.indexed to decouple parsing and RAG further.
- **6.4** Documentation: runbooks, architecture diagrams, README for the new repo.
- **6.5** Monitoring: health checks, metrics, and logs per service; gateway and each service observable.

**Checkpoint**: System is fully microservices in the new repo; docs and ops updated.

---

## 3. Checkpoints and Rollback

- **Checkpoint** at end of each phase: deploy and test; gate next phase on green tests and sign-off.
- **Rollback**: Gateway routing is configurable (env or config). To roll back a service, fix or revert that service in the new repo; no monolith in this repo to fall back to. If you still run the old RAGFlow app separately, you could point the gateway at it for specific paths during transition.
- **Data**: Prefer reversible schema and data changes until a phase is stable.

---

## 4. Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Inconsistent auth across services | identity-service as single source of JWT; gateway or services validate and pass user/tenant headers. |
| Document/parsing/RAG ordering and failures | document-service orchestrates; idempotent parsing and indexing; retries and dead-letter queue. |
| Cross-service transactions | Avoid; use eventual consistency, compensating actions, clear ownership per entity. |
| Frontend breaking when changing routes | Gateway preserves same path and response shape; version API if contract changes. |
| Performance (extra hop via gateway) | Keep gateway thin; HTTP/2 and connection pooling. |
| Porting errors from RAGFlow | Use RAGFlow repo as reference only; test each service against expected API and behavior.

---

## Document Links

- [Index](./INDEX.md)
- [High-level overview](./HIGH_LEVEL_OVERVIEW.md)
- [Monorepo structure](./MONOREPO_STRUCTURE.md)
- [Database per service](./DATABASE_PER_SERVICE.md)
- [Docker strategy](./DOCKER_STRATEGY.md)
