# System Architecture Overview

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            Clients                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│  │  Web UI     │  │  Mobile     │  │  API        │  │  LLM Tool   │   │
│  │  (React)    │  │  (React N.) │  │  (Direct)   │  │  (MCP)      │   │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘   │
└─────────┼────────────────┼────────────────┼────────────────┼───────────┘
          │                │                │                │
          └────────────────┴────────┬───────┴────────────────┘
                                    │
                          ┌─────────▼─────────┐
                          │    API Gateway    │
                          │  (Rate Limiting,  │
                          │   Auth, Routing)  │
                          └─────────┬─────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          │                         │                         │
┌─────────▼─────────┐    ┌─────────▼─────────┐    ┌─────────▼─────────┐
│   Chat Service    │    │  Search Service   │    │  Tool Service     │
│  (Conversations)  │    │  (RAG Pipeline)   │    │  (Drug/Lit API)   │
└─────────┬─────────┘    └─────────┬─────────┘    └─────────┬─────────┘
          │                        │                        │
          └────────────────────────┼────────────────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
           ┌────────▼────────┐ ┌───▼────┐ ┌──────▼──────┐
           │  Vector Store   │ │ Cache  │ │  External   │
           │ (Chroma/Pinecone│ │(Redis) │ │    APIs     │
           └─────────────────┘ └────────┘ │  PubMed     │
                                          │  RxNorm     │
                                          │  FDA        │
                                          └─────────────┘
```

---

## Core Components

### 1. API Layer
- **Purpose**: REST/WEBSOCKET endpoints for all clients
- **Options**: FastAPI (Python) or Express (Node.js)
- **Features**: Rate limiting, authentication, request validation

### 2. Chat Service
- **Purpose**: Manage conversation history, context
- **State**: Conversation threads, user preferences
- **Storage**: PostgreSQL (production) or in-memory (dev)

### 3. Search Service (RAG Pipeline)
- **Purpose**: Process queries through retrieval + generation
- **Flow**: Query → Retrieve → Augment → Generate → Cite
- **Framework**: LangChain / LangGraph

### 4. Tool Service
- **Purpose**: Provide specialized tools (drug lookup, literature search)
- **Integration**: RxNorm API, PubMed E-utilities, FDA openFDA

### 5. Vector Store
- **Purpose**: Store embedded medical literature
- **Options**: Chroma (dev), Pinecone (cloud), pgvector (self-hosted)

---

## Data Flow Examples

### Chat Flow
```
User: "What are the side effects of Metformin?"

1. API receives request
2. Chat Service creates/updates conversation
3. Search Service:
   a. Rewrite query ("Metformin adverse reactions")
   b. Retrieve top-k chunks from vector store
   c. Build prompt with context + retrieved docs
   d. Call LLM with prompt
   e. Extract citations from retrieved docs
4. Generate response with sources
5. Return streaming response to client
```

### Tool Call Flow (Drug Interaction)
```
User: "Does Aspirin interact with Warfarin?"

1. Agent recognizes tool need
2. Tool Service calls RxNorm API
3. Get RxCUIs for both drugs
4. Query interaction endpoint
5. Return formatted interaction info
```

---

## Component Responsibilities

| Component | Responsibility | State |
|-----------|----------------|-------|
| API Layer | HTTP, WebSocket, Auth | Stateless |
| Chat Service | Conversations, history | Stateful |
| Search Service | RAG pipeline execution | Stateless |
| Tool Service | External API calls | Stateless |
| Vector Store | Document storage/retrieval | Persistent |

---

## Deployment Options

### Development
```
API (localhost:8000) → Chroma (local) → OpenAI API
```

### Production Cloud
```
API (Docker/K8s) → Pinecone → OpenAI API
                   → PostgreSQL
                   → Redis (cache)
```

### Self-Hosted
```
API (Docker) → pgvector → Ollama (local LLM)
                        → PostgreSQL
                        → Redis
```

---

## See Also

- [API Layer Options](./api-layer-options.md)
- [RAG Pipeline Architecture](./rag-pipeline.md)
- [Data Storage](./data-storage.md)
- [Tech Stack Selection](./tech-stack-selection.md)