# RAGFlow Golden Path

## Overview

The golden path represents the ideal user journey through RAGFlow, from initial setup to getting AI-powered answers from your documents.

> **📊 Viewing Diagrams**: This document contains Mermaid diagrams. If they don't render in your preview:
> - Copy the Mermaid code and paste it into [Mermaid Live Editor](https://mermaid.live/)
> - Use the ASCII/text diagrams provided below each Mermaid diagram
> - View in GitHub/GitLab which have native Mermaid support

## System Architecture

> **💡 Tip**: If diagrams don't render, view them online at [Mermaid Live Editor](https://mermaid.live/) or use the ASCII diagrams below.

```mermaid
graph TB
    subgraph "Frontend Layer"
        UI[React/TypeScript UI]
    end
    
    subgraph "API Layer"
        API[Quart/Flask API Server]
        KB_API[Knowledge Base API]
        DOC_API[Document API]
        CHAT_API[Chat API]
    end
    
    subgraph "Service Layer"
        KB_SVC[KnowledgebaseService]
        DOC_SVC[DocumentService]
        FILE_SVC[FileService]
        CHAT_SVC[DialogService]
    end
    
    subgraph "Processing Layer"
        PARSER[Document Parser]
        EMBED[Embedding Model]
        RAG[RAG Retrieval]
        LLM[LLM Chat Model]
    end
    
    subgraph "Data Layer"
        DB[(MySQL/PostgreSQL)]
        REDIS[(Redis)]
        ES[(Elasticsearch/Infinity)]
        STORAGE[(MinIO/S3)]
    end
    
    UI --> API
    API --> KB_API
    API --> DOC_API
    API --> CHAT_API
    
    KB_API --> KB_SVC
    DOC_API --> DOC_SVC
    DOC_API --> FILE_SVC
    CHAT_API --> CHAT_SVC
    
    DOC_SVC --> PARSER
    PARSER --> EMBED
    EMBED --> ES
    
    CHAT_SVC --> RAG
    RAG --> ES
    RAG --> LLM
    
    KB_SVC --> DB
    DOC_SVC --> DB
    FILE_SVC --> STORAGE
    CHAT_SVC --> DB
    
    DOC_SVC --> REDIS
    RAG --> REDIS
```

**ASCII Diagram:**
```
┌─────────────────────────────────────────────────────────┐
│              Frontend Layer (React/TypeScript)          │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│              API Layer (Quart/Flask)                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  KB API      │  │  DOC API    │  │  CHAT API    │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
└─────────┼─────────────────┼──────────────────┼──────────┘
          │                 │                  │
          ▼                 ▼                  ▼
┌─────────────────────────────────────────────────────────┐
│              Service Layer                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ KB Service   │  │ DOC Service  │  │ CHAT Service │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
└─────────┼─────────────────┼──────────────────┼──────────┘
          │                 │                  │
          ▼                 ▼                  ▼
┌─────────────────────────────────────────────────────────┐
│              Processing Layer                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Parser     │  │  Embedding   │  │  RAG/LLM     │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
└─────────┼─────────────────┼──────────────────┼──────────┘
          │                 │                  │
          ▼                 ▼                  ▼
┌─────────────────────────────────────────────────────────┐
│              Data Layer                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   MySQL      │  │ Elasticsearch│  │  MinIO/S3    │  │
│  │   Redis      │  │              │  │              │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## Golden Path Flow

### Step 1: User Authentication

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant API
    participant AuthService
    participant DB
    
    User->>Frontend: Access RAGFlow
    Frontend->>API: POST /api/v1/user/login
    API->>AuthService: Validate credentials
    AuthService->>DB: Check user
    DB-->>AuthService: User data
    AuthService-->>API: JWT token
    API-->>Frontend: Authentication success
    Frontend-->>User: Redirect to home
```

**Flow:**
```
User → Frontend → API → AuthService → Database
                                    ↓
User ← Frontend ← API ← AuthService ← (JWT Token)
```

**Key Endpoints:**
- `POST /api/v1/user/login` - User login
- `POST /api/v1/user/register` - User registration

### Step 2: Create Knowledge Base

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant API
    participant KBService
    participant DB
    
    User->>Frontend: Click "Create Dataset"
    Frontend->>API: POST /api/v1/datasets
    Note over Frontend,API: {name, embedding_model, parser_type}
    API->>KBService: create()
    KBService->>DB: Insert Knowledgebase record
    DB-->>KBService: KB ID
    KBService-->>API: Knowledge base created
    API-->>Frontend: KB details
    Frontend-->>User: Show KB configuration page
```

**Flow:**
```
User clicks "Create Dataset"
    ↓
Frontend sends: {name, embedding_model, parser_type}
    ↓
API → KBService → Database (creates KB record)
    ↓
Returns KB ID → Frontend → User sees configuration page
```

**Key Endpoints:**
- `POST /api/v1/datasets` - Create knowledge base
- `GET /api/v1/datasets/:id` - Get KB details

**Configuration Required:**
- **Embedding Model**: Converts text to vectors (e.g., `bge-large-en-v1.5`)
- **Parser Type**: Chunking strategy (e.g., `naive`, `paper`, `book`)
- **Language**: Document language

### Step 3: Upload Documents

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant API
    participant FileService
    participant DocService
    participant Storage
    participant TaskQueue
    
    User->>Frontend: Upload file(s)
    Frontend->>API: POST /api/v1/documents/upload
    Note over Frontend,API: multipart/form-data with file
    API->>FileService: upload_document()
    FileService->>Storage: Save file to MinIO/S3
    Storage-->>FileService: File URL
    FileService->>DocService: create()
    DocService->>TaskQueue: Queue parsing task
    TaskQueue-->>API: Task queued
    API-->>Frontend: Document created, parsing started
    Frontend-->>User: Show upload progress
```

**Flow:**
```
User uploads file
    ↓
Frontend → API (multipart/form-data)
    ↓
FileService → Storage (MinIO/S3) → saves file
    ↓
DocService → creates document record
    ↓
TaskQueue → queues parsing task
    ↓
User sees "Processing..." status
```

**Key Endpoints:**
- `POST /api/v1/documents/upload` - Upload document
- `GET /api/v1/documents/list?kb_id=:id` - List documents

**Supported Formats:**
- PDF, DOCX, PPTX, XLSX
- Images (PNG, JPG) with OCR
- Text files, Markdown
- Web URLs (web crawling)

### Step 4: Document Processing

```mermaid
sequenceDiagram
    participant TaskQueue
    participant TaskExecutor
    participant Parser
    participant Embedding
    participant SearchEngine
    
    TaskQueue->>TaskExecutor: Consume task
    TaskExecutor->>Parser: Parse document
    Note over Parser: Extract text, images, tables
    Parser-->>TaskExecutor: Chunks
    TaskExecutor->>Embedding: Generate embeddings
    Embedding-->>TaskExecutor: Vectors
    TaskExecutor->>SearchEngine: Index chunks
    SearchEngine-->>TaskExecutor: Indexed
    TaskExecutor->>TaskQueue: Update progress
    TaskQueue-->>User: Processing complete
```

**Flow:**
```
TaskQueue → TaskExecutor (consumes task)
    ↓
Parser → extracts text/images/tables → creates chunks
    ↓
Embedding Model → generates vector embeddings
    ↓
Elasticsearch → indexes chunks with vectors
    ↓
Status updated → User sees "Complete"
```

**Processing Steps:**
1. **Parse**: Extract content based on parser type
2. **Chunk**: Split into semantic chunks
3. **Embed**: Generate vector embeddings
4. **Index**: Store in Elasticsearch/Infinity

**Key Components:**
- `rag/svr/task_executor.py` - Task processor
- `deepdoc/parser/` - Document parsers
- `rag/llm/embedding_model.py` - Embedding generation

### Step 5: Create Chat Assistant

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant API
    participant DialogService
    participant DB
    
    User->>Frontend: Create new chat
    Frontend->>API: POST /api/v1/chats
    Note over Frontend,API: {dataset_ids, llm_id, prompt_config}
    API->>DialogService: create()
    DialogService->>DB: Insert Dialog record
    DB-->>DialogService: Dialog ID
    DialogService-->>API: Dialog created
    API-->>Frontend: Chat ready
    Frontend-->>User: Show chat interface
```

**Flow:**
```
User creates new chat
    ↓
Frontend sends: {dataset_ids, llm_id, prompt_config}
    ↓
API → DialogService → Database (creates dialog)
    ↓
Returns chat ID → Frontend → User sees chat interface
```

**Key Endpoints:**
- `POST /api/v1/chats` - Create chat session
- `GET /api/v1/chats` - List chats

**Configuration:**
- **Knowledge Bases**: Select one or more datasets
- **LLM Model**: Choose chat model (e.g., GPT-4, Claude)
- **Prompt Template**: Customize system prompt
- **Retrieval Settings**: Top-K, similarity threshold

### Step 6: Ask Questions & Get Answers

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant API
    participant DialogService
    participant Retriever
    participant SearchEngine
    participant LLM
    participant ChatService
    
    User->>Frontend: Type question
    Frontend->>API: POST /api/v1/conversations/ask
    Note over Frontend,API: {question, kb_ids}
    API->>DialogService: Get dialog config
    DialogService->>Retriever: retrieval()
    Retriever->>SearchEngine: Vector + Fulltext search
    SearchEngine-->>Retriever: Relevant chunks
    Retriever->>Retriever: Rerank (optional)
    Retriever-->>DialogService: Top chunks
    DialogService->>LLM: Build prompt with context
    LLM-->>DialogService: Stream response
    DialogService->>ChatService: Save conversation
    DialogService-->>API: Stream answer
    API-->>Frontend: SSE stream
    Frontend-->>User: Display answer with citations
```

**Flow:**
```
User types question
    ↓
Frontend → API (POST /conversations/ask)
    ↓
DialogService → Retriever
    ↓
Retriever → Elasticsearch (vector + keyword search)
    ↓
Returns relevant chunks → (optional rerank)
    ↓
LLM → generates answer with context
    ↓
Streams response (SSE) → Frontend → User sees answer
```

**Key Endpoints:**
- `POST /api/v1/conversations/ask` - Ask question (SSE stream)
- `GET /api/v1/conversations/:id` - Get conversation history

**RAG Process:**
1. **Query Embedding**: Convert question to vector
2. **Retrieval**: Hybrid search (vector + keyword)
3. **Reranking**: Optional LLM-based reranking
4. **Context Building**: Format chunks for LLM
5. **Generation**: LLM generates answer with citations

## Complete Golden Path Diagram

```mermaid
flowchart TD
    Start([User Starts]) --> Login[Login/Register]
    Login --> Home[Home Dashboard]
    Home --> CreateKB[Create Knowledge Base]
    CreateKB --> ConfigKB[Configure KB<br/>- Embedding Model<br/>- Parser Type<br/>- Language]
    ConfigKB --> Upload[Upload Documents]
    Upload --> Process[Document Processing<br/>- Parse<br/>- Chunk<br/>- Embed<br/>- Index]
    Process --> Wait{Processing<br/>Complete?}
    Wait -->|No| Wait
    Wait -->|Yes| CreateChat[Create Chat Assistant]
    CreateChat --> ConfigChat[Configure Chat<br/>- Select KBs<br/>- Choose LLM<br/>- Set Prompt]
    ConfigChat --> Ask[Ask Questions]
    Ask --> Retrieve[RAG Retrieval]
    Retrieve --> Generate[LLM Generation]
    Generate --> Answer[Get Answer with Citations]
    Answer --> Ask
    Answer --> End([Success!])
    
    style Start fill:#e1f5ff
    style End fill:#d4edda
    style Process fill:#fff3cd
    style Retrieve fill:#f8d7da
    style Generate fill:#d1ecf1
```

**Complete Flow (Text):**
```
┌─────────────────────────────────────────────────────────┐
│                    GOLDEN PATH FLOW                      │
└─────────────────────────────────────────────────────────┘

1. User Starts
   │
   ▼
2. Login/Register
   │
   ▼
3. Home Dashboard
   │
   ▼
4. Create Knowledge Base
   │
   ▼
5. Configure KB
   │   ├─ Embedding Model
   │   ├─ Parser Type
   │   └─ Language
   │
   ▼
6. Upload Documents
   │
   ▼
7. Document Processing
   │   ├─ Parse (extract content)
   │   ├─ Chunk (split into pieces)
   │   ├─ Embed (generate vectors)
   │   └─ Index (store in search engine)
   │
   ▼
8. Wait for Processing Complete
   │   └─ Check status periodically
   │
   ▼
9. Create Chat Assistant
   │
   ▼
10. Configure Chat
    │   ├─ Select Knowledge Bases
    │   ├─ Choose LLM Model
    │   └─ Set Prompt Template
    │
    ▼
11. Ask Questions
    │
    ▼
12. RAG Retrieval
    │   ├─ Vector search
    │   ├─ Keyword search
    │   └─ Rerank results
    │
    ▼
13. LLM Generation
    │   └─ Generate answer with context
    │
    ▼
14. Get Answer with Citations
    │
    ▼
15. Success! (Can ask more questions)
```

## Key User Actions

| Step | Action | Endpoint | Result |
|------|--------|-----------|--------|
| 1 | Login | `POST /api/v1/user/login` | JWT token |
| 2 | Create KB | `POST /api/v1/datasets` | Knowledge base ID |
| 3 | Upload Doc | `POST /api/v1/documents/upload` | Document ID |
| 4 | Wait Processing | `GET /api/v1/documents/:id` | Check status |
| 5 | Create Chat | `POST /api/v1/chats` | Chat ID |
| 6 | Ask Question | `POST /api/v1/conversations/ask` | Streaming answer |

## Data Flow Summary

```
User Input
    ↓
Frontend (React)
    ↓
API Layer (Quart/Flask)
    ↓
Service Layer (Business Logic)
    ↓
┌─────────────┬──────────────┬─────────────┐
│   Storage   │   Database   │   Search    │
│  (MinIO/S3) │  (MySQL/ES)  │ (Elasticsearch)│
└─────────────┴──────────────┴─────────────┘
```

## Quick Start Checklist

- [ ] Start RAGFlow services (Docker or source)
- [ ] Access web UI (default: http://localhost:8000)
- [ ] Register/Login
- [ ] Create a knowledge base
- [ ] Configure embedding model and parser
- [ ] Upload documents
- [ ] Wait for processing to complete
- [ ] Create a chat assistant
- [ ] Ask your first question!

## Next Steps

After completing the golden path:
- Explore **Agents** for complex workflows
- Set up **Data Source Connectors** for automated sync
- Configure **Teams** for collaboration
- Use **Memory** for agent context
- Customize **Prompts** for better responses

---

*This golden path represents the core workflow. For advanced features, refer to the [full documentation](https://ragflow.io/docs/).*
