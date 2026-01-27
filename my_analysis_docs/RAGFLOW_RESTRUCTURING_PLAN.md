# RAGFlow Multi-Stack Restructuring Plan

## Executive Summary

This plan outlines an incremental migration strategy to restructure RAGFlow from a monolithic Python application into a multi-stack architecture supporting Python, Node.js, and .NET. The approach maintains backward compatibility, enables parallel development, and provides clear feature boundaries for team allocation.

## 1. Feature Inventory & Classification

### 1.1 Core Features Identified

Based on codebase analysis, RAGFlow consists of the following major feature domains:

#### **A. User Management & Authentication** (`api/apps/user_app.py`, `api/apps/auth/`)

- User registration/login
- OAuth/OIDC (GitHub, Feishu)
- Password reset (OTP-based)
- User settings & profile management
- Session management
- API token management

#### **B. Knowledge Base Management** (`api/apps/kb_app.py`, `api/apps/document_app.py`)

- Knowledge base CRUD operations
- Document upload & parsing
- Chunk management & visualization
- Document metadata management
- Knowledge graph generation

#### **C. Document Processing** (`deepdoc/`, `rag/flow/`)

- Multi-format parsing (PDF, DOCX, Excel, PPT, images, etc.)
- OCR capabilities
- Template-based chunking
- Document layout analysis
- Vision model integration

#### **D. RAG & Retrieval** (`rag/`, `api/apps/search_app.py`)

- Vector embedding generation
- Semantic search
- Multi-recall with re-ranking
- Cross-language query support
- Retrieval evaluation

#### **E. Chat & Conversation** (`api/apps/conversation_app.py`, `api/apps/dialog_app.py`)

- Chat session management
- Message history
- Streaming responses
- Related questions generation
- Chat sharing & widgets

#### **F. Agent System** (`agent/`, `api/apps/canvas_app.py`)

- Agent workflow orchestration
- Canvas-based agent builder
- Agent templates
- Tool integration (MCP, plugins)
- Code execution (sandbox)
- Memory management

#### **G. Data Source Connectors** (`common/data_source/`)

- 20+ integrations: S3, Notion, Confluence, Discord, Google Drive, Gmail, Jira, GitHub, GitLab, Slack, Teams, WebDAV, Moodle, Dropbox, Box, Airtable, Asana, Bitbucket, Zendesk, IMAP, OCI Storage, R2, Google Cloud Storage
- Data synchronization
- Rate limiting & retry logic

#### **H. LLM Management** (`api/apps/llm_app.py`, `rag/llm/`)

- LLM factory management
- API key configuration
- Model abstraction layer
- Embedding & rerank models
- TTS & sequence-to-text models

#### **I. File Management** (`api/apps/file_app.py`)

- File upload/download
- Folder structure
- File conversion
- Attachment handling

#### **J. Evaluation & Testing** (`api/apps/evaluation_app.py`)

- Dataset management
- Test case creation
- Evaluation execution
- Results analysis
- Configuration recommendations

#### **K. System Administration** (`api/apps/system_app.py`, `api/apps/tenant_app.py`)

- System health monitoring
- Tenant management
- Service status
- Configuration management
- Admin APIs

#### **L. SDK & API** (`api/apps/sdk/`)

- RESTful API endpoints
- OpenAI-compatible endpoints
- Webhook support
- API versioning

### 1.2 Feature Dependencies Map

```
User Management → All Features (Auth dependency)
Knowledge Base → Document Processing → RAG & Retrieval
Chat → RAG & Retrieval → Knowledge Base
Agent System → LLM Management → RAG & Retrieval → Knowledge Base
Data Connectors → Document Processing → Knowledge Base
Evaluation → RAG & Retrieval → Knowledge Base
```

## 2. Technology Stack Recommendations

### 2.1 Node.js Recommended For

**Rationale**: Excellent for I/O-intensive operations, real-time features, and rapid API development

1. **User Management & Authentication**
   - Fast JWT handling
   - OAuth library ecosystem
   - Session management with Redis

2. **File Management**
   - Stream handling
   - File upload/download optimization
   - Integration with cloud storage SDKs

3. **Chat & Conversation**
   - WebSocket/SSE support
   - Real-time streaming
   - Event-driven architecture

4. **Data Source Connectors** (Most)
   - Rich npm ecosystem for integrations
   - Async/await patterns
   - Rate limiting libraries

5. **SDK & API Layer**
   - Express/Fastify for REST APIs
   - OpenAPI/Swagger tooling
   - API gateway patterns

6. **System Administration**
   - Health check endpoints
   - Monitoring integration
   - Configuration management

### 2.2 .NET Recommended For

**Rationale**: Strong typing, enterprise features, performance for CPU-intensive tasks

1. **Document Processing** (Core parsing logic)
   - Strong typing for complex data structures
   - Performance for CPU-intensive parsing
   - Integration with ML.NET if needed

2. **RAG & Retrieval Engine**
   - Vector operations performance
   - Memory management for large datasets
   - Parallel processing capabilities

3. **Evaluation & Testing**
   - Strong test framework ecosystem
   - Data processing pipelines
   - Statistical analysis

4. **Agent System** (Core orchestration)
   - Complex workflow management
   - State machine patterns
   - Enterprise-grade reliability

5. **LLM Management** (Abstraction layer)
   - Interface-based design
   - Dependency injection
   - Configuration management

### 2.3 Python Remains For

**Rationale**: Existing codebase, ML/AI libraries, team expertise

1. **DeepDoc** (`deepdoc/`)
   - OCR models (PaddleOCR, etc.)
   - Vision models
   - Layout recognition
   - Complex ML pipelines

2. **Advanced RAG Features** (`rag/advanced_rag/`)
   - Tree-structured query decomposition
   - RAPTOR implementation
   - Specialized retrieval algorithms

3. **Agent Tools** (`agent/tools/`)
   - Python code execution
   - Sandbox management
   - Specialized tool implementations

4. **Legacy Integrations**
   - Existing connectors not migrated
   - Python-specific libraries

## 3. Architecture Design

### 3.1 Target Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Frontend (React)                      │
│              (No changes required)                        │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│              API Gateway / Load Balancer                 │
│         (Routes to appropriate backend service)           │
└─────────────────────────────────────────────────────────┘
         │              │              │
         ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   Node.js    │ │     .NET     │ │   Python     │
│   Services   │ │   Services   │ │   Services   │
└──────────────┘ └──────────────┘ └──────────────┘
         │              │              │
         └──────────────┴──────────────┘
                          │
         ┌────────────────┼────────────────┐
         ▼                ▼                 ▼
    ┌─────────┐    ┌──────────┐    ┌──────────┐
    │  MySQL  │    │  Redis   │    │  MinIO   │
    │  (Shared)│    │  (Shared)│    │  (Shared)│
    └─────────┘    └──────────┘    └──────────┘
```

### 3.2 Service Communication Strategy

**Option 1: HTTP REST APIs (Recommended for start)**
- Each service exposes REST endpoints
- API Gateway routes requests
- Simple, well-understood pattern

**Option 2: Message Queue (For async operations)**
- RabbitMQ/Kafka for document processing
- Event-driven architecture
- Better for long-running tasks

**Option 3: gRPC (For high-performance)**
- Inter-service communication
- Type-safe contracts
- Better performance than REST

### 3.3 Shared Resources

- **Database**: MySQL/PostgreSQL (shared schema, service-specific tables)
- **Cache**: Redis (shared for sessions, rate limiting)
- **Storage**: MinIO/S3 (shared for files)
- **Search**: Elasticsearch/Infinity (shared for vector search)

## 4. Incremental Migration Strategy

### Phase 1: Foundation (Weeks 1-4)

**Goal**: Set up infrastructure and migrate one simple feature

#### Week 1-2: Infrastructure Setup

- [ ] Set up API Gateway (Nginx/Envoy/Kong)
- [ ] Create shared database schema documentation
- [ ] Set up service discovery/configuration
- [ ] Create inter-service authentication mechanism
- [ ] Set up CI/CD pipelines for Node.js and .NET
- [ ] Create API contract documentation standards

#### Week 3-4: First Feature Migration - User Management (Node.js)

**Why Start Here**: 
- Low dependency on other features
- Well-defined boundaries
- High visibility

**Tasks**:
- [ ] Create Node.js service structure
- [ ] Migrate user registration/login endpoints
- [ ] Implement JWT authentication
- [ ] Migrate OAuth handlers
- [ ] Set up database access layer
- [ ] Create API compatibility layer
- [ ] Update API Gateway routing
- [ ] Comprehensive testing
- [ ] Gradual traffic migration (10% → 50% → 100%)

**Files to Reference**:
- `api/apps/user_app.py`
- `api/db/services/user_service.py`
- `api/db/db_models.py` (User, Tenant models)
- `api/apps/auth/`

### Phase 2: File & Document Management (Weeks 5-8)

#### Week 5-6: File Management (Node.js)

- [ ] Migrate file upload/download endpoints
- [ ] Implement folder structure management
- [ ] File conversion service
- [ ] Integration with MinIO/S3

**Files to Reference**:
- `api/apps/file_app.py`
- `api/db/services/file_service.py`

#### Week 7-8: Document App Endpoints (Node.js)

- [ ] Document CRUD operations
- [ ] Document metadata management
- [ ] Integration with Python DeepDoc service (via HTTP)

**Files to Reference**:
- `api/apps/document_app.py`
- `api/db/services/document_service.py`

### Phase 3: Core RAG Features (Weeks 9-16)

#### Week 9-12: RAG Engine (.NET)

- [ ] Vector embedding generation service
- [ ] Semantic search implementation
- [ ] Re-ranking service
- [ ] Integration with Elasticsearch/Infinity

**Files to Reference**:
- `rag/nlp/search.py`
- `rag/llm/embedding_model.py`
- `rag/llm/rerank_model.py`
- `api/apps/search_app.py`

#### Week 13-16: Knowledge Base Management (Node.js)

- [ ] Knowledge base CRUD
- [ ] Chunk management
- [ ] Chunk visualization
- [ ] Integration with RAG engine

**Files to Reference**:
- `api/apps/kb_app.py`
- `api/apps/chunk_app.py`
- `api/db/services/knowledgebase_service.py`

### Phase 4: Chat & Conversation (Weeks 17-20)

#### Week 17-20: Chat System (Node.js)

- [ ] Chat session management
- [ ] Message history
- [ ] Streaming response handling (SSE/WebSocket)
- [ ] Related questions generation
- [ ] Integration with RAG engine

**Files to Reference**:
- `api/apps/conversation_app.py`
- `api/apps/dialog_app.py`
- `api/apps/sdk/session.py`

### Phase 5: Agent System (Weeks 21-28)

#### Week 21-24: Agent Core (.NET)

- [ ] Agent workflow orchestration
- [ ] Canvas data model
- [ ] State management
- [ ] Tool integration framework

**Files to Reference**:
- `agent/canvas.py`
- `agent/component/`
- `api/apps/canvas_app.py`

#### Week 25-28: Agent Tools & Execution (Python + Node.js)

- [ ] Keep Python for code execution sandbox
- [ ] Migrate tool wrappers to Node.js
- [ ] MCP server integration (Node.js)
- [ ] Memory management (Node.js)

**Files to Reference**:
- `agent/tools/`
- `api/apps/mcp_server_app.py`
- `memory/services/`

### Phase 6: Data Connectors (Weeks 29-36)

#### Week 29-36: Connector Migration (Node.js)

- [ ] Migrate connectors one by one
- [ ] Start with high-priority: S3, Notion, Confluence
- [ ] Implement rate limiting & retry logic
- [ ] Data synchronization service

**Files to Reference**:
- `common/data_source/`
- `api/apps/connector_app.py`

### Phase 7: Advanced Features (Weeks 37-44)

#### Week 37-40: Evaluation System (.NET)

- [ ] Dataset management
- [ ] Test execution engine
- [ ] Results analysis
- [ ] Configuration recommendations

**Files to Reference**:
- `api/apps/evaluation_app.py`
- `api/db/services/evaluation_service.py`

#### Week 41-44: LLM Management (Node.js)

- [ ] LLM factory management
- [ ] API key configuration
- [ ] Model abstraction layer
- [ ] Integration with embedding/rerank services

**Files to Reference**:
- `api/apps/llm_app.py`
- `rag/llm/`

### Phase 8: SDK & System Services (Weeks 45-48)

#### Week 45-48: Final Migrations

- [ ] SDK endpoint consolidation
- [ ] System administration endpoints
- [ ] Health monitoring
- [ ] Performance optimization
- [ ] Documentation completion

## 5. Integration Patterns

### 5.1 Database Access Strategy

**Shared Schema Approach**:
- All services access the same database
- Service-specific tables for new features
- Shared tables for common entities (User, Tenant)
- Use database views/stored procedures for complex queries

**Implementation**:
- Node.js: Use Sequelize/TypeORM/Prisma
- .NET: Use Entity Framework Core
- Python: Keep existing Peewee ORM

### 5.2 Inter-Service Communication

**Synchronous (HTTP REST)**:
```typescript
// Node.js calling .NET RAG service
const response = await fetch('http://rag-service/api/v1/search', {
  method: 'POST',
  headers: { 'Authorization': `Bearer ${token}` },
  body: JSON.stringify({ query, dataset_id })
});
```

**Asynchronous (Message Queue)**:
```csharp
// .NET publishing document processing event
await messageBus.PublishAsync(new DocumentProcessedEvent {
  DocumentId = docId,
  Status = "completed"
});
```

### 5.3 Authentication & Authorization

**Shared JWT Tokens**:
- All services validate same JWT format
- Shared secret/key for token validation
- User context passed in headers

**Implementation**:
- Node.js: `jsonwebtoken`, `express-jwt`
- .NET: `Microsoft.AspNetCore.Authentication.JwtBearer`
- Python: Keep existing implementation

### 5.4 Error Handling & Logging

**Centralized Logging**:
- All services log to same system (ELK, Loki, etc.)
- Correlation IDs for request tracing
- Structured logging format

**Error Propagation**:
- Standardized error response format
- Error codes mapping
- Graceful degradation

## 6. Testing Strategy

### 6.1 Unit Testing
- Each service has comprehensive unit tests
- Mock external dependencies
- Test coverage > 80%

### 6.2 Integration Testing
- Test inter-service communication
- Database integration tests
- End-to-end API tests

### 6.3 Contract Testing
- API contract tests (Pact, etc.)
- Ensure backward compatibility
- Version compatibility testing

### 6.4 Performance Testing
- Load testing for each service
- Latency benchmarks
- Resource usage monitoring

## 7. Risk Mitigation

### 7.1 Backward Compatibility
- Maintain Python API endpoints during migration
- Feature flags for gradual rollout
- Canary deployments

### 7.2 Data Consistency
- Database transactions where needed
- Eventual consistency patterns
- Data migration scripts

### 7.3 Team Coordination
- Clear API contracts
- Regular sync meetings
- Shared documentation
- Code review process

### 7.4 Rollback Strategy
- Feature flags for instant rollback
- Database migration rollback scripts
- Service versioning

## 8. Success Metrics

- **Functionality**: 100% feature parity
- **Performance**: < 10% latency increase
- **Reliability**: 99.9% uptime maintained
- **Code Quality**: Maintain test coverage
- **Team Velocity**: No significant slowdown

## 9. Documentation Requirements

- [ ] API documentation (OpenAPI/Swagger)
- [ ] Architecture decision records (ADRs)
- [ ] Service runbooks
- [ ] Database schema documentation
- [ ] Deployment guides
- [ ] Integration guides

## 10. Next Steps

1. **Review & Approve Plan**: Get stakeholder buy-in
2. **Team Allocation**: Assign Node.js and .NET developers
3. **Infrastructure Setup**: Begin Phase 1
4. **POC**: Build proof-of-concept for first feature
5. **Iterate**: Begin incremental migration

---

**Key Files for Reference**:
- API Routes: `api/apps/__init__.py`, all `*_app.py` files
- Database Models: `api/db/db_models.py`
- Services: `api/db/services/`
- Core Logic: `rag/`, `deepdoc/`, `agent/`
- Frontend Routes: `web/src/routes.tsx`

## Migration Phases Summary

| Phase | Duration | Features | Tech Stack | Priority |
|-------|----------|----------|------------|----------|
| Phase 1 | Weeks 1-4 | Infrastructure + User Management | Node.js | Critical |
| Phase 2 | Weeks 5-8 | File & Document Management | Node.js | High |
| Phase 3 | Weeks 9-16 | RAG Engine + Knowledge Base | .NET + Node.js | Critical |
| Phase 4 | Weeks 17-20 | Chat & Conversation | Node.js | High |
| Phase 5 | Weeks 21-28 | Agent System | .NET + Node.js + Python | High |
| Phase 6 | Weeks 29-36 | Data Connectors | Node.js | Medium |
| Phase 7 | Weeks 37-44 | Evaluation + LLM Management | .NET + Node.js | Medium |
| Phase 8 | Weeks 45-48 | SDK & System Services | All | Low |

**Total Estimated Duration**: 48 weeks (~12 months)
