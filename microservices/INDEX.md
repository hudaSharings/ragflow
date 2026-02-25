# RAGFlow Monolith → Microservices — Documentation Index

Quick navigation to all microservices decomposition documents. **No implementation has been started**; these are planning documents for your review and confirmation.

**Important:** The monorepo is a **new, dedicated repository**. It does **not** contain the existing RAGFlow monolith code. Restructuring is done by building microservices and an Angular frontend in this new repo; the current RAGFlow codebase remains a separate reference only.

---

## 1. Documents Overview

| Document | Purpose |
|----------|---------|
| [**HIGH_LEVEL_OVERVIEW.md**](./HIGH_LEVEL_OVERVIEW.md) | Decomposition strategy, proposed microservices, monorepo layout, design principles |
| [**STEP_BY_STEP_PLAN.md**](./STEP_BY_STEP_PLAN.md) | Phased migration plan with ordered steps and checkpoints |
| [**MONOREPO_STRUCTURE.md**](./MONOREPO_STRUCTURE.md) | Target directory structure (backend services, frontend Angular) |
| [**DATABASE_PER_SERVICE.md**](./DATABASE_PER_SERVICE.md) | **Database per service**: table → service mapping, per-service DBs, cross-service references |
| [**DOCKER_STRATEGY.md**](./DOCKER_STRATEGY.md) | Use of base compose, microservices compose, and runtime topology |

---

## 2. Quick Links by Topic

### Strategy & Scope
- [High-level decomposition strategy](./HIGH_LEVEL_OVERVIEW.md#2-decomposition-strategy)
- [Proposed microservices list](./HIGH_LEVEL_OVERVIEW.md#3-proposed-microservices)
- [Service boundaries and responsibilities](./HIGH_LEVEL_OVERVIEW.md#4-service-boundaries-and-responsibilities)

### Structure
- [Monorepo layout](./MONOREPO_STRUCTURE.md#1-monorepo-layout)
- [Backend services (Node/Java/Go)](./MONOREPO_STRUCTURE.md#2-backend-services)
- [Frontend (Angular)](./MONOREPO_STRUCTURE.md#3-frontend-angular)
- [**Database per service & table mapping**](./DATABASE_PER_SERVICE.md#3-table--service-mapping)

### Execution
- [Phase overview](./STEP_BY_STEP_PLAN.md#1-phase-overview)
- [Step-by-step plan](./STEP_BY_STEP_PLAN.md#2-step-by-step-plan)
- [Docker: base vs microservices compose](./DOCKER_STRATEGY.md#1-compose-files)

### Operations
- [API gateway and routing](./HIGH_LEVEL_OVERVIEW.md#5-api-gateway--routing)
- [Data and integration considerations](./HIGH_LEVEL_OVERVIEW.md#6-data-and-integration-considerations)
- [Risks and mitigations](./STEP_BY_STEP_PLAN.md#4-risks-and-mitigations)

---

## 3. Current Monolith (Reference Only — Not in This Repo)

The existing RAGFlow application lives in a **separate repository**. It is used only as a **reference** for behavior, API contracts, and data models when porting logic into the new microservices. The new monorepo does **not** include or run any monolith code.

- **Backend (reference)**: Python (Quart/Flask), `api/` + `rag/` + `deepdoc/` + `agent/`
- **Frontend (reference)**: React + TypeScript + UmiJS in `web/`
- **Infrastructure (reference)**: `docker-compose-base.yml` (MySQL, Redis, MinIO, ES/Infinity, etc.) — **copied or adapted** into the new repo’s `docker/`; no monolith container in the new repo.
- **API**: REST under `/v1/<module>` and `/api/v1/<sdk>` (same paths preserved by the new gateway and services)

---

## 4. Next Steps

1. Create the **new dedicated repository** and adopt the layout in [MONOREPO_STRUCTURE.md](./MONOREPO_STRUCTURE.md) (no monolith code).
2. Review [HIGH_LEVEL_OVERVIEW.md](./HIGH_LEVEL_OVERVIEW.md) and confirm service boundaries and tech choices.
3. Review [STEP_BY_STEP_PLAN.md](./STEP_BY_STEP_PLAN.md) and adjust phases/steps as needed.
4. Confirm [DOCKER_STRATEGY.md](./DOCKER_STRATEGY.md) (base + microservices compose live in the new repo).
5. After confirmation, implementation proceeds in the new repo according to the step-by-step plan.
