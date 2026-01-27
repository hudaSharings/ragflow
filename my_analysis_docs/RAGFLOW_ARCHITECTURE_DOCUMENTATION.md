# RAGFlow Current Architecture & Codebase Documentation

## Purpose

This document provides a comprehensive understanding of RAGFlow's existing architecture, data flows, and codebase structure. Use this alongside the restructuring plan to understand how the system currently works before migration.

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Architecture Layers](#2-architecture-layers)
3. [Core Workflows](#3-core-workflows)
4. [Data Models & Database Schema](#4-data-models--database-schema)
5. [Service Layer Architecture](#5-service-layer-architecture)
6. [Key Components Deep Dive](#6-key-components-deep-dive)
7. [Data Flow Diagrams](#7-data-flow-diagrams)
8. [Integration Points](#8-integration-points)

---

## 1. System Overview

### 1.1 High-Level Architecture

RAGFlow is a monolithic Python application built with:
- **Backend Framework**: Quart (async Flask)
- **Frontend**: React + TypeScript + UmiJS
- **Database**: MySQL/PostgreSQL (Peewee ORM)
- **Cache**: Redis
- **Storage**: MinIO/S3
- **Search Engine**: Elasticsearch or Infinity
- **Task Queue**: Redis Streams

### 1.2 Main Components

```
┌─────────────────────────────────────────────────────────┐
│                    Frontend (React)                      │
│  - Pages: Chat, Knowledge Base, Agents, Settings        │
│  - Routes: web/src/routes.tsx                           │
└─────────────────────────────────────────────────────────┘
                          │ HTTP/REST
                          ▼
┌─────────────────────────────────────────────────────────┐
│              API Layer (Quart/Flask)                     │
│  - api/apps/*_app.py (Route handlers)                  │
│  - api/utils/ (Helpers, validation)                     │
└─────────────────────────────────────────────────────────┘
                          │
         ┌────────────────┼────────────────┐
         ▼                ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Service   │  │   RAG Core   │  │  DeepDoc     │
│   Layer     │  │   (rag/)     │  │  (deepdoc/)  │
│ (api/db/    │  │              │  │              │
│  services/) │  │ - Search     │  │ - Parsers    │
│             │  │ - Embedding  │  │ - OCR        │
│ - User      │  │ - Rerank     │  │ - Vision     │
│ - KB        │  │ - Advanced   │  │              │
│ - Document  │  │   RAG        │  │              │
│ - Chat      │  └──────────────┘  └──────────────┘
│ - Agent     │
└──────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│              Data Layer                                  │
│  - MySQL/PostgreSQL (via Peewee ORM)                    │
│  - Redis (Cache, Sessions, Task Queue)                  │
│  - MinIO/S3 (File Storage)                              │
│  - Elasticsearch/Infinity (Vector Search)               │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Architecture Layers

### 2.1 API Layer (`api/apps/`)

**Purpose**: HTTP request handling, routing, validation

**Key Files**:
- `api/apps/__init__.py`: App initialization, authentication middleware
- `api/apps/*_app.py`: Feature-specific route handlers
- `api/utils/api_utils.py`: Response helpers, error handling

**Pattern**: Blueprint-based routing
```python
# Example from api/apps/kb_app.py
@manager.route('/create', methods=['post'])
@login_required
@validate_request("name")
async def create():
    # Handler logic
```

**Authentication Flow**:
1. Request comes with `Authorization` header
2. `_load_user()` in `api/apps/__init__.py` validates JWT/API token
3. `@login_required` decorator enforces authentication
4. `current_user` provides user context

### 2.2 Service Layer (`api/db/services/`)

**Purpose**: Business logic, database operations, data transformation

**Pattern**: Service classes extend `CommonService` base class

**Key Services**:
- `UserService`: User management, authentication
- `KnowledgebaseService`: Knowledge base CRUD
- `DocumentService`: Document lifecycle, parsing orchestration
- `FileService`: File upload/download, storage
- `ConversationService`: Chat session management
- `DialogService`: Dialog configuration
- `LLMService`: LLM model management
- `TaskService`: Background task management

**Service Pattern**:
```python
class DocumentService(CommonService):
    model = Document  # Peewee model
    
    @classmethod
    def create(cls, **kwargs):
        # Business logic + database operation
        return cls.model.insert(**kwargs)
```

### 2.3 Data Layer (`api/db/`)

**Models** (`api/db/db_models.py`):
- `User`: User accounts, authentication
- `Tenant`: Multi-tenancy support
- `Knowledgebase`: Knowledge base/dataset
- `Document`: Document metadata
- `File`: File storage metadata
- `Chunk`: Document chunks for RAG
- `Conversation`: Chat sessions
- `Dialog`: Dialog configurations
- `Task`: Background processing tasks

**Database Connection**:
- Uses Peewee ORM with connection pooling
- Supports MySQL and PostgreSQL
- Retry logic for connection failures

### 2.4 RAG Core (`rag/`)

**Components**:
- `rag/nlp/search.py`: Search/retrieval logic
- `rag/llm/`: LLM abstractions (chat, embedding, rerank)
- `rag/app/`: Parser-specific chunking strategies
- `rag/flow/`: Dataflow/pipeline components
- `rag/advanced_rag/`: Advanced retrieval algorithms
- `rag/utils/`: Connections (ES, Redis, storage)

### 2.5 Document Processing (`deepdoc/`)

**Components**:
- `deepdoc/parser/`: Format-specific parsers (PDF, DOCX, Excel, etc.)
- `deepdoc/vision/`: OCR, layout recognition, image processing

---

## 3. Core Workflows

### 3.1 Document Upload & Processing Workflow

```
User Uploads File
       │
       ▼
┌──────────────────┐
│ FileService      │
│ - Save to MinIO  │
│ - Create File    │
│   record in DB   │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│ DocumentService  │
│ - Create Document│
│   record         │
│ - Link to KB     │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│ TaskService      │
│ - Queue task to  │
│   Redis Streams  │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│ task_executor.py  │
│ - Consume task   │
│ - Parse document │
│ - Generate chunks│
│ - Create vectors │
│ - Index to ES     │
└──────────────────┘
```

**Key Code Locations**:
- Upload: `api/apps/document_app.py::upload_and_parse()`
- Task Queue: `api/db/services/task_service.py::queue_tasks()`
- Processing: `rag/svr/task_executor.py::do_handle_task()`
- Parsing: `rag/app/*.py` (format-specific parsers)

### 3.2 RAG Retrieval Workflow

```
User Query
    │
    ▼
┌──────────────────┐
│ SearchService    │
│ - Validate query │
│ - Get KB config  │
└──────────────────┘
    │
    ▼
┌──────────────────┐
│ Dealer.retrieval │
│ (rag/nlp/search) │
│ - Embed query    │
│ - Vector search  │
│ - Fulltext search│
│ - Hybrid ranking │
└──────────────────┘
    │
    ▼
┌──────────────────┐
│ Rerank (optional)│
│ - Use rerank LLM │
│ - Reorder results│
└──────────────────┘
    │
    ▼
┌──────────────────┐
│ Return chunks    │
│ with metadata    │
└──────────────────┘
```

**Key Code Locations**:
- Search endpoint: `api/apps/search_app.py`
- Retrieval logic: `rag/nlp/search.py::Dealer.retrieval()`
- Embedding: `rag/llm/embedding_model.py`
- Rerank: `rag/llm/rerank_model.py`

### 3.3 Chat/Conversation Workflow

```
User sends message
    │
    ▼
┌──────────────────┐
│ ConversationApp  │
│ - Create message │
│   record         │
└──────────────────┘
    │
    ▼
┌──────────────────┐
│ RAG Retrieval    │
│ - Get context    │
│   from KB        │
└──────────────────┘
    │
    ▼
┌──────────────────┐
│ LLM Chat         │
│ - Build prompt   │
│ - Call LLM API   │
│ - Stream response│
└──────────────────┘
    │
    ▼
┌──────────────────┐
│ Save response    │
│ - Store message  │
│ - Update session │
└──────────────────┘
```

**Key Code Locations**:
- Chat endpoint: `api/apps/conversation_app.py`
- Session management: `api/apps/sdk/session.py`
- LLM chat: `rag/llm/chat_model.py`

### 3.4 Agent Execution Workflow

```
Agent triggered
    │
    ▼
┌──────────────────┐
│ Canvas execution │
│ - Load agent     │
│   definition     │
└──────────────────┘
    │
    ▼
┌──────────────────┐
│ Component chain  │
│ - Execute nodes  │
│ - Handle state   │
│ - Call tools     │
└──────────────────┘
    │
    ▼
┌──────────────────┐
│ Tool execution   │
│ - RAG retrieval  │
│ - Code execution │
│ - External APIs  │
└──────────────────┘
```

**Key Code Locations**:
- Agent execution: `agent/canvas.py`
- Components: `agent/component/`
- Tools: `agent/tools/`

---

## 4. Data Models & Database Schema

### 4.1 Core Entities

**User & Tenant**:
```python
User
├── id (PK)
├── email
├── password_hash
├── access_token
├── tenant_id (FK → Tenant)
└── status

Tenant
├── id (PK)
├── name
└── settings (JSON)
```

**Knowledge Base**:
```python
Knowledgebase
├── id (PK)
├── name
├── tenant_id (FK)
├── embd_id (embedding model)
├── llm_id (LLM model)
├── parser_id (chunking strategy)
├── language
└── settings (JSON)
```

**Document & Chunk**:
```python
Document
├── id (PK)
├── name
├── kb_id (FK → Knowledgebase)
├── parser_id
├── status
└── progress

Chunk (stored in Elasticsearch/Infinity)
├── id
├── doc_id (FK → Document)
├── kb_id
├── content_with_weight
├── vector (embedding)
├── page_number
└── metadata (JSON)
```

**Conversation & Dialog**:
```python
Conversation
├── id (PK)
├── dialog_id (FK → Dialog)
├── user_id (FK → User)
└── messages (JSON)

Dialog
├── id (PK)
├── name
├── kb_ids (list of KB IDs)
├── llm_id
└── prompt_config (JSON)
```

### 4.2 Database Access Pattern

**Peewee ORM Usage**:
```python
# Query example
users = UserService.query(email=email, status=StatusEnum.VALID.value)

# Create
user = UserService.create(email=email, password=hashed_pw)

# Update
UserService.update_by_id(user_id, **updates)
```

**Connection Management**:
- Uses `@DB.connection_context()` decorator
- Connection pooling via `PooledMySQLDatabase`
- Automatic retry on connection failures

---

## 5. Service Layer Architecture

### 5.1 CommonService Base Class

All services extend `CommonService` which provides:
- CRUD operations (create, read, update, delete)
- Query building
- Pagination
- Error handling

**Location**: `api/db/services/common_service.py`

### 5.2 Service Responsibilities

| Service | Responsibilities |
|---------|------------------|
| `UserService` | Authentication, user CRUD, password management |
| `KnowledgebaseService` | KB CRUD, configuration, indexing setup |
| `DocumentService` | Document lifecycle, parsing orchestration, progress tracking |
| `FileService` | File upload/download, MinIO integration, blob management |
| `ConversationService` | Chat session management, message history |
| `DialogService` | Dialog configuration, prompt management |
| `TaskService` | Background task queue, task status tracking |
| `LLMService` | LLM model management, API key storage |
| `SearchService` | Search configuration, saved searches |
| `ConnectorService` | Data source connector management, sync scheduling |

### 5.3 Service Interaction Pattern

Services can call other services:
```python
# Example: DocumentService calling KnowledgebaseService
e, kb = KnowledgebaseService.get_by_id(kb_id)
if not e:
    raise LookupError("Knowledge base not found")
```

---

## 6. Key Components Deep Dive

### 6.1 Document Parser System

**Location**: `rag/app/` and `deepdoc/parser/`

**Parser Types**:
- `naive.py`: General text chunking
- `paper.py`: Academic paper parsing
- `book.py`: Book chapter parsing
- `presentation.py`: PPT parsing
- `table.py`: Spreadsheet parsing
- `picture.py`: Image OCR
- `email.py`: Email parsing
- `audio.py`: Audio transcription

**Parser Interface**:
```python
def chunk(name, blob, callback, parser_config, **kwargs):
    # Parse document
    # Return list of chunks
    return chunks
```

**Chunk Structure**:
```python
{
    "content_with_weight": "text content",
    "page_number": 1,
    "image": <PIL Image or None>,
    "metadata": {...}
}
```

### 6.2 RAG Search System

**Location**: `rag/nlp/search.py`

**Dealer Class**:
- Handles hybrid search (vector + fulltext)
- Supports multiple recall strategies
- Implements re-ranking
- Handles pagination and aggregation

**Search Flow**:
1. Embed query using embedding model
2. Vector similarity search in Elasticsearch
3. Fulltext search (BM25)
4. Combine results with hybrid scoring
5. Apply re-ranking if configured
6. Return top-k chunks

### 6.3 LLM Abstraction Layer

**Location**: `rag/llm/`

**LLMBundle Class** (`api/db/services/llm_service.py`):
- Unified interface for different LLM providers
- Supports: OpenAI, Anthropic, local models, etc.
- Handles API keys, rate limiting, retries

**Model Types**:
- `ChatModel`: Text generation
- `EmbeddingModel`: Vector embeddings
- `RerankModel`: Result re-ranking
- `OCRModel`: Image text extraction
- `TTSModel`: Text-to-speech

### 6.4 Task Execution System

**Location**: `rag/svr/task_executor.py`

**Task Types**:
- `parse`: Document parsing
- `dataflow`: Pipeline execution
- `raptor`: RAPTOR processing
- `graphrag`: Knowledge graph generation
- `memory`: Memory storage

**Task Queue**:
- Uses Redis Streams
- Priority-based processing
- Progress tracking
- Cancellation support

**Task Structure**:
```python
{
    "id": "task_id",
    "doc_id": "document_id",
    "kb_id": "knowledgebase_id",
    "task_type": "parse",
    "parser_config": {...},
    "from_page": 0,
    "to_page": 100
}
```

### 6.5 Agent System

**Location**: `agent/`

**Components**:
- `canvas.py`: Agent definition and execution
- `component/`: Reusable agent components
- `tools/`: Agent tools (RAG, code execution, etc.)
- `templates/`: Pre-built agent templates

**Component Types**:
- `LLM`: LLM calls
- `RAG`: Knowledge retrieval
- `Loop`: Iteration control
- `Switch`: Conditional logic
- `VariableAssigner`: State management
- `CodeExecutor`: Python/JS execution

---

## 7. Data Flow Diagrams

### 7.1 Document Processing Flow

```
┌─────────────┐
│   Upload    │
│   Endpoint  │
└──────┬──────┘
       │
       ▼
┌─────────────┐      ┌─────────────┐
│ FileService │─────▶│   MinIO/S3   │
│ (Save file) │      │  (Storage)   │
└──────┬──────┘      └─────────────┘
       │
       ▼
┌─────────────┐
│DocumentService│
│ (Create doc) │
└──────┬──────┘
       │
       ▼
┌─────────────┐      ┌─────────────┐
│ TaskService │─────▶│ Redis Stream│
│ (Queue task)│      │  (Task Queue)│
└─────────────┘      └──────┬──────┘
                            │
                            ▼
                    ┌─────────────┐
                    │task_executor│
                    │ (Consumer)  │
                    └──────┬──────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                  ▼
    ┌─────────┐    ┌──────────┐    ┌──────────┐
    │ Parser  │───▶│ Embedding│───▶│Elasticsearch│
    │(deepdoc)│    │  Model   │    │  (Index)   │
    └─────────┘    └──────────┘    └──────────┘
```

### 7.2 Chat Flow

```
┌─────────────┐
│ User Query  │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│Conversation │
│   Service   │
│(Save query) │
└──────┬──────┘
       │
       ▼
┌─────────────┐      ┌─────────────┐
│   RAG       │─────▶│Elasticsearch│
│ Retrieval   │      │  (Search)   │
└──────┬──────┘      └─────────────┘
       │
       ▼
┌─────────────┐
│ Build Prompt│
│ (with ctx)  │
└──────┬──────┘
       │
       ▼
┌─────────────┐      ┌─────────────┐
│   LLM API   │─────▶│  OpenAI/    │
│   Call      │      │  Anthropic  │
└──────┬──────┘      └─────────────┘
       │
       ▼
┌─────────────┐
│ Stream      │
│ Response    │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Save Message│
│ (to DB)     │
└─────────────┘
```

### 7.3 Agent Execution Flow

```
┌─────────────┐
│ Agent Start │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Load Canvas │
│ Definition  │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Execute     │
│ Components  │
│ (Sequential)│
└──────┬──────┘
       │
       ├─────────┐
       │         │
       ▼         ▼
┌─────────┐ ┌─────────┐
│  LLM    │ │   RAG    │
│Component│ │Component│
└────┬────┘ └────┬────┘
     │           │
     └─────┬─────┘
           │
           ▼
    ┌─────────────┐
    │   Result    │
    │  Aggregation│
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │   Output    │
    └─────────────┘
```

---

## 8. Integration Points

### 8.1 External Services

**LLM Providers**:
- OpenAI API
- Anthropic API
- Local models (Ollama, etc.)
- Custom endpoints

**Storage**:
- MinIO (default)
- AWS S3
- Azure Blob Storage
- Google Cloud Storage

**Search Engines**:
- Elasticsearch (default)
- Infinity (alternative)
- OpenSearch

**Data Source Connectors**:
- 20+ integrations (S3, Notion, Confluence, etc.)
- OAuth-based authentication
- Rate limiting and retry logic

### 8.2 Internal Communication

**Synchronous**:
- Direct function calls between services
- Shared database access

**Asynchronous**:
- Redis Streams for task queue
- Event-driven patterns for document processing

### 8.3 API Endpoints Structure

**Base URL**: `/api/v1/` or `/{version}/{feature}`

**Feature Endpoints**:
- `/user/*`: User management
- `/knowledgebase/*`: KB operations
- `/document/*`: Document operations
- `/conversation/*`: Chat operations
- `/search/*`: Search operations
- `/agent/*`: Agent operations
- `/file/*`: File operations
- `/llm/*`: LLM configuration
- `/system/*`: System administration

**SDK Endpoints** (`/api/v1/`):
- `/chats/*`: Chat SDK
- `/agents/*`: Agent SDK
- `/datasets/*`: Dataset SDK
- `/sessions/*`: Session SDK

---

## 9. Key Design Patterns

### 9.1 Service Pattern
- Business logic in service classes
- Database access abstracted
- Reusable across API endpoints

### 9.2 Factory Pattern
- Parser factory (`FACTORY` dict in `task_executor.py`)
- LLM factory (`LLMBundle`)

### 9.3 Strategy Pattern
- Different chunking strategies per document type
- Multiple retrieval strategies (vector, fulltext, hybrid)

### 9.4 Observer Pattern
- Progress callbacks in document processing
- Event handlers in agent system

---

## 10. Configuration Management

### 10.1 Settings

**Location**: `common/settings.py`

**Key Settings**:
- Database connection
- Redis connection
- Storage configuration
- LLM API keys
- Feature flags

### 10.2 Runtime Configuration

**Location**: `api/db/runtime_config.py`

**Purpose**: Dynamic configuration that can be updated without restart

---

## 11. Error Handling

### 11.1 Error Response Format

```python
{
    "code": RetCode.SUCCESS,
    "message": "success",
    "data": {...}
}
```

### 11.2 Error Codes

Defined in `common/constants.py`:
- `SUCCESS`: Operation successful
- `AUTHENTICATION_ERROR`: Auth failure
- `OPERATING_ERROR`: Business logic error
- `SERVER_ERROR`: Internal server error

---

## 12. Testing Structure

**Location**: `test/`

**Test Types**:
- Unit tests for services
- Integration tests for APIs
- End-to-end tests for workflows

---

## 13. Migration Considerations

When migrating features, pay attention to:

1. **Database Dependencies**: Services share database tables
2. **Service Dependencies**: Services call each other directly
3. **Shared State**: Redis cache, session storage
4. **File Storage**: MinIO/S3 access patterns
5. **Search Integration**: Elasticsearch/Infinity queries
6. **Task Queue**: Redis Streams consumer groups
7. **Authentication**: JWT token validation
8. **Configuration**: Settings and runtime config

---

## 14. Key Files Reference

### API Layer
- `api/ragflow_server.py`: Application entry point
- `api/apps/__init__.py`: App initialization, auth middleware
- `api/apps/*_app.py`: Route handlers for each feature

### Service Layer
- `api/db/services/common_service.py`: Base service class
- `api/db/services/*_service.py`: Feature-specific services

### Data Layer
- `api/db/db_models.py`: Database models
- `api/db/db_utils.py`: Database utilities

### RAG Core
- `rag/nlp/search.py`: Search/retrieval logic
- `rag/llm/`: LLM abstractions
- `rag/app/`: Parser implementations
- `rag/svr/task_executor.py`: Background task processor

### Document Processing
- `deepdoc/parser/`: Document parsers
- `deepdoc/vision/`: OCR and vision models

### Agent System
- `agent/canvas.py`: Agent execution engine
- `agent/component/`: Reusable components
- `agent/tools/`: Agent tools

---

## Conclusion

This document provides a foundation for understanding RAGFlow's current architecture. Use it alongside the restructuring plan to:

1. Understand existing patterns before migration
2. Identify integration points between services
3. Plan data migration strategies
4. Design API contracts for new services
5. Ensure feature parity during migration

For specific implementation details, refer to the source code files listed in each section.
