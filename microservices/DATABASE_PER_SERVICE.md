# Database per Service — Decomposition and Mapping

This document describes **database-per-service** for the new monorepo: each microservice owns its data and uses its own database (or dedicated schema). It maps the current RAGFlow monolithic schema to services and explains how to split and reference data across boundaries.

---

## Table of Contents

1. [Principle: Database per Service](#1-principle-database-per-service)
2. [Current RAGFlow Tables (Reference)](#2-current-ragflow-tables-reference)
3. [Table → Service Mapping](#3-table--service-mapping)
4. [Per-Service Database Summary](#4-per-service-database-summary)
5. [Cross-Service References and No Shared FKs](#5-cross-service-references-and-no-shared-fks)
6. [Deployment Options: One MySQL vs Multiple](#6-deployment-options-one-mysql-vs-multiple)
7. [Migration Path from Monolith Schema](#7-migration-path-from-monolith-schema)

---

## 1. Principle: Database per Service

- **Each microservice has its own database** (or dedicated schema within a shared MySQL instance). No service connects to another service’s database.
- **Service owns its data**: only that service’s code reads/writes its tables. Others get data via API or events.
- **No shared foreign keys** across services: references use IDs (e.g. `tenant_id`, `kb_id`, `document_id`). Referential integrity is not enforced across DBs.
- **RAGFlow today**: single MySQL DB, all tables in one schema, many FKs and joins. We decompose by **ownership** and **bounded context** into one DB (or schema) per service.

---

## 2. Current RAGFlow Tables (Reference)

From `api/db/db_models.py` (monolith). All tables live in one database today.

| # | Table | Model | Purpose |
|---|--------|--------|----------|
| 1 | `user` | User | User accounts, auth, profile |
| 2 | `tenant` | Tenant | Tenant (workspace) config, default LLM/embedding IDs |
| 3 | `user_tenant` | UserTenant | User–tenant membership, roles |
| 4 | `invitation_code` | InvitationCode | Tenant invitation codes |
| 5 | `llm_factories` | LLMFactories | LLM factory definitions (e.g. OpenAI, Ollama) |
| 6 | `llm` | LLM | LLM/model registry (per factory) |
| 7 | `tenant_llm` | TenantLLM | Tenant-specific LLM config, API keys |
| 8 | `tenant_langfuse` | TenantLangfuse | Langfuse config per tenant |
| 9 | `knowledgebase` | Knowledgebase | Knowledge base (dataset) metadata and config |
| 10 | `document` | Document | Document metadata, parsing/indexing progress |
| 11 | `file` | File | File/folder metadata, storage location |
| 12 | `file2document` | File2Document | Link file ↔ document |
| 13 | `task` | Task | Document processing tasks (chunking, etc.) |
| 14 | `dialog` | Dialog | Dialog (chat app) configuration |
| 15 | `conversation` | Conversation | Chat conversations, messages |
| 16 | `api_token` | APIToken | API tokens (tenant/dialog/agent scoped) |
| 17 | `api_4_conversation` | API4Conversation | API conversation history |
| 18 | `user_canvas` | UserCanvas | Agent/dataflow canvas definitions |
| 19 | `canvas_template` | CanvasTemplate | Canvas templates |
| 20 | `user_canvas_version` | UserCanvasVersion | Canvas version history |
| 21 | `mcp_server` | MCPServer | MCP server config per tenant |
| 22 | `search` | Search | Saved search configurations |
| 23 | `pipeline_operation_log` | PipelineOperationLog | Document pipeline run logs |
| 24 | `connector` | Connector | Data source connector configs |
| 25 | `connector2kb` | Connector2Kb | Connector–KB association |
| 26 | `sync_logs` | SyncLogs | Connector sync run logs |
| 27 | `evaluation_datasets` | EvaluationDataset | RAG evaluation datasets |
| 28 | `evaluation_cases` | EvaluationCase | Evaluation test cases |
| 29 | `evaluation_runs` | EvaluationRun | Evaluation run metadata |
| 30 | `evaluation_results` | EvaluationResult | Per-case evaluation results |
| 31 | `memory` | Memory | Memory (semantic/episodic) config and metadata |
| 32 | `system_settings` | SystemSettings | System-wide settings |

**Note:** Chunk/vector data in RAGFlow lives in **Elasticsearch or Infinity**, not in MySQL. The **rag-service** owns that store; no chunk table in MySQL per service.

---

## 3. Table → Service Mapping

Each table is assigned to **exactly one** service. That service is the only one that reads/writes it.

| Service | Tables | Notes |
|---------|--------|--------|
| **identity-service** | `user`, `tenant`, `user_tenant`, `invitation_code`, `api_token` | Auth, tenants, membership, API tokens |
| **llm-gateway-service** | `llm_factories`, `llm`, `tenant_llm`, `tenant_langfuse` | LLM/model registry and tenant model config |
| **knowledge-base-service** | `knowledgebase` | KB metadata and config only; documents/chunks are in other services |
| **document-service** | `document`, `file2document`, `task`, `pipeline_operation_log` | Document lifecycle, file–document link, processing tasks and logs |
| **file-service** | `file` | File/folder metadata and storage references (MinIO paths) |
| **rag-service** | *(none in MySQL)* | Uses Elasticsearch/Infinity for chunks and vectors; no relational tables required (or optional small table for job status only) |
| **chat-service** | `dialog`, `conversation`, `api_4_conversation` | Dialog config, conversations, API conversation history |
| **search-service** | `search` | Saved search configurations |
| **agent-service** | `user_canvas`, `canvas_template`, `user_canvas_version` | Canvas (agent/dataflow) definitions and versions |
| **connector-service** | `connector`, `connector2kb`, `sync_logs` | Connectors and sync state |
| **admin-service** | `system_settings`, `mcp_server` | System settings; MCP server config (admin/tenant config) |
| **memory** (optional service or part of chat/agent) | `memory` | Memory config and metadata; if you add a dedicated memory-service, it owns this table |
| **evaluation** (optional service or part of admin) | `evaluation_datasets`, `evaluation_cases`, `evaluation_runs`, `evaluation_results` | RAG evaluation; can be a separate service or folded into admin-service |

---

## 4. Per-Service Database Summary

| Service | Database name (example) | Tables count | Stores |
|---------|-------------------------|--------------|--------|
| identity-service | `identity_db` | 5 | user, tenant, user_tenant, invitation_code, api_token |
| llm-gateway-service | `llm_gateway_db` | 4 | llm_factories, llm, tenant_llm, tenant_langfuse |
| knowledge-base-service | `knowledge_base_db` | 1 | knowledgebase |
| document-service | `document_db` | 4 | document, file2document, task, pipeline_operation_log |
| file-service | `file_db` | 1 | file |
| rag-service | — | 0 (or 1) | ES/Infinity only; optional job table in MySQL |
| chat-service | `chat_db` | 3 | dialog, conversation, api_4_conversation |
| search-service | `search_db` | 1 | search |
| agent-service | `agent_db` | 3 | user_canvas, canvas_template, user_canvas_version |
| connector-service | `connector_db` | 3 | connector, connector2kb, sync_logs |
| admin-service | `admin_db` | 2 | system_settings, mcp_server |
| memory (if separate) | `memory_db` | 1 | memory |
| evaluation (if separate) | `evaluation_db` | 4 | evaluation_* |

---

## 5. Cross-Service References and No Shared FKs

- **References only by ID**: Services store `tenant_id`, `user_id`, `kb_id`, `document_id`, `dialog_id`, etc. as plain strings. They do **not** create foreign keys to another service’s database.
- **Resolve via API**: If identity-service needs to validate `tenant_id`, it uses its own DB. If document-service needs to check “does this kb_id exist?”, it can call knowledge-base-service API (or trust the gateway/validation).
- **Consistency**: Eventual. When a tenant is deleted in identity-service, other services may still have rows referencing that `tenant_id` until you run cleanup jobs or listen for “tenant.deleted” events.
- **Duplicate/cache**: Services may cache minimal data (e.g. tenant name) from another service’s API; no shared DB access.

---

## 6. Deployment Options: One MySQL vs Multiple

**Option A — Single MySQL instance, multiple databases**

- One MySQL server; create one database per service (e.g. `identity_db`, `document_db`, …).
- Each service connects only to its own database (different `db_name` in connection config).
- Simpler to operate; same backup/user management as today. Good for getting started.

**Option B — Multiple MySQL instances (or DB per service on different hosts)**

- Each service gets its own MySQL instance (or dedicated RDS/Cloud SQL).
- Stronger isolation and independent scaling. More operational overhead.

Recommendation: Start with **Option A** (one MySQL, multiple databases). Move to Option B per service when you need isolation or scaling.

---

## 7. Migration Path from Monolith Schema

- **New repo, new schemas**: When implementing each service, create **new** tables in its own database (same schema as current RAGFlow tables, or normalized/cleaned). Do not point the new services at the monolith’s single DB.
- **Data migration**: If you need to migrate existing data from the current RAGFlow DB:
  1. Export data per table group (per service).
  2. Import into the corresponding service database in the new repo’s environment.
  3. Run services against the new DBs; no monolith in the new repo.
- **Order**: Migrate identity and tenant data first (identity-service DB), then file/document, then chat, etc., so that IDs (tenant_id, user_id) exist when other services reference them.
- **Chunks/vectors**: rag-service uses ES/Infinity; if you already have an index from the monolith, point rag-service at the same cluster (or reindex from document-service + rag-service). No MySQL chunk table to split.

---

## Document Links

- [Index](./INDEX.md)
- [High-level overview](./HIGH_LEVEL_OVERVIEW.md)
- [Step-by-step plan](./STEP_BY_STEP_PLAN.md)
- [Monorepo structure](./MONOREPO_STRUCTURE.md)
- [Docker strategy](./DOCKER_STRATEGY.md)
