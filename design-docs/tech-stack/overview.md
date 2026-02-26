# Tech Stack Documentation

## Overview

This document outlines the recommended technology stack for the Medical Knowledge Assistant, with detailed coverage of each component including vector databases, LLMs, embedding models, and infrastructure.

---

## 1. Core Technology Stack Summary

### Recommended Stack

| Layer | Technology | Justification |
|-------|------------|---------------|
| **API Framework** | FastAPI | Native LangChain support, async, auto-docs |
| **RAG Framework** | LangChain / LangGraph | Best-in-class RAG implementation |
| **LLM (Cloud)** | OpenAI GPT-4 / Claude 3.5 | High medical accuracy |
| **LLM (Self-hosted)** | Ollama / vLLM | Privacy, cost control |
| **Embeddings** | text-embedding-3-large | 3072 dimensions, medical precision |
| **Vector DB (Dev)** | Chroma | Easy local development |
| **Vector DB (Prod)** | pgvector / Pinecone | Production-grade |
| **Relational DB** | PostgreSQL | User data, conversations |
| **Cache** | Redis | Query caching, rate limiting |
| **Task Queue** | Celery | Background jobs |

---

## 2. Chroma Vector Database (Detailed)

### Overview

Chroma is an open-source vector database designed for AI applications. It's particularly well-suited for development and prototyping phases.

- **GitHub**: https://github.com/chroma-core/chroma
- **Website**: https://trychroma.com
- **License**: Apache 2.0
- **Python Support**: 3.8+

### Key Features

1. **Embedded Mode** - Runs in-process, no separate server
2. **Client-Server Mode** - Deployable as standalone service
3. **Metadata Filtering** - Filter by source, year, document type
4. **Hierarchical Navigable Small World (HNSW)** - Fast approximate nearest neighbor search
5. **Automatic Persistence** - Data saved to disk automatically
6. **LangChain Integration** - Native support via `langchain-chroma`

### Installation

```bash
pip install chromadb langchain-chroma
```

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Application                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │   LangChain │    │   Chroma    │    │  Embeddings │     │
│  │    Agent    │───▶│   Client    │───▶│   Model     │     │
│  └─────────────┘    └──────┬──────┘    └─────────────┘     │
└─────────────────────────────┼────────────────────────────────┘
                              │
                     ┌────────▼────────┐
                     │   Chroma DB    │
                     │  (Local File)  │
                     │                 │
                     │ - index/        │
                     │ - metadata.db   │
                     │ - embeddings/   │
                     └─────────────────┘
```

### Basic Usage

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings
from langchain_core.documents import Document

# Initialize with persistence
vector_store = Chroma(
    collection_name="medical_knowledge",
    embedding_function=OpenAIEmbeddings(
        model="text-embedding-3-large",
        dimensions=3072
    ),
    persist_directory="./data/chroma_db"
)

# Add documents with metadata
documents = [
    Document(
        page_content="Metformin is the first-line medication for treating type 2 diabetes mellitus.",
        metadata={
            "source": "rxnorm",
            "drug_name": "Metformin",
            "category": "antidiabetic",
            "pmid": "12345678"
        }
    ),
    Document(
        page_content="Dengue fever is a mosquito-borne tropical disease caused by the dengue virus.",
        metadata={
            "source": "pubmed",
            "disease": "dengue",
            "pmid": "87654321"
        }
    )
]

vector_store.add_documents(documents=documents)

# Similarity search
results = vector_store.similarity_search(
    query="first line treatment for diabetes",
    k=3,
    filter={"source": "rxnorm"}  # Optional metadata filter
)

for doc in results:
    print(f"Content: {doc.page_content[:100]}...")
    print(f"Source: {doc.metadata.get('source')}")
    print("---")

# Search with similarity scores
results_with_scores = vector_store.similarity_search_with_score(
    query="diabetes medication",
    k=5
)

for doc, score in results_with_scores:
    print(f"Score: {score:.4f} | Content: {doc.page_content[:80]}...")
```

### Collection Management

```python
# Get existing collection
collection = vector_store.get()

# Delete collection
vector_store.delete_collection()

# Create new collection with custom settings
from chromadb.config import Settings

vector_store = Chroma(
    collection_name="medical_documents",
    embedding_function=embeddings,
    persist_directory="./chroma_db",
    client_settings=Settings(
        anonymized_telemetry=False,
        allow_reset=True,
    )
)
```

### Metadata Filtering

Chroma supports rich metadata filtering using where clauses:

```python
# Filter by single metadata field
results = vector_store.similarity_search(
    query="diabetes treatment",
    k=5,
    filter={"source": "pubmed"}
)

# Filter by multiple fields (AND)
results = vector_store.similarity_search(
    query="hypertension management",
    k=5,
    filter={
        "$and": [
            {"source": {"$eq": "guidelines"}},
            {"year": {"$gte": 2020}}
        ]
    }
)

# Filter with OR logic
results = vector_store.similarity_search(
    query="sepsis management",
    k=5,
    filter={
        "$or": [
            {"source": {"$eq": "moh_cpg"}},
            {"source": {"$eq": "pubmed"}}
        ]
    }
)

# Complex filters
results = vector_store.similarity_search(
    query="antimicrobial guidelines",
    k=5,
    filter={
        "$and": [
            {"source": {"$in": ["moh_cpg", "guidelines"]}},
            {"year": {"$gte": 2019}},
            {"category": {"$ne": "archived"}}
        ]
    }
)
```

### Retriever with Filters

```python
from langchain_core.runnables import RunnablePassthrough

# Create retriever with filters
retriever = vector_store.as_retriever(
    search_type="similarity",
    search_kwargs={
        "k": 5,
        "filter": {
            "source": {"$in": ["pubmed", "guidelines", "rxnorm"]}
        }
    }
)

# Use in RAG chain
rag_chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt_template
    | llm
)
```

### Indexing for Performance

```python
# Enable HNSW indexing for faster search
# Note: This is done automatically by Chroma

# Check collection stats
collection = vector_store.get()
print(f"Total documents: {collection.get('total_documents', 0)}")
print(f"Collection name: {collection.get('name')}")

# Force index rebuild after bulk additions
vector_store.persist()
```

### Data Persistence Structure

```
chroma_db/
├── index/
│   └── (Chroma's internal HNSW index files)
├── metadata.db
│   └── SQLite database for metadata
└── embeddings/
    └── (Embedding files)
```

### Production Considerations

While Chroma is excellent for development, consider these for production:

1. **Data Backup**: Regular backup of persist_directory
2. **Disk Space**: Monitor embeddings storage growth
3. **Memory**: Load only necessary collections
4. **Updates**: Use versioning for embeddings

```python
# Backup function
import shutil
import os
from datetime import datetime

def backup_chroma_db(source_dir: str, backup_dir: str):
    """Backup Chroma database."""
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    backup_path = os.path.join(backup_dir, f"chroma_backup_{timestamp}")
    shutil.copytree(source_dir, backup_path)
    print(f"Backup created: {backup_path}")

# Restore function
def restore_chroma_db(backup_dir: str, target_dir: str):
    """Restore Chroma database from backup."""
    if os.path.exists(target_dir):
        shutil.rmtree(target_dir)
    shutil.copytree(backup_dir, target_dir)
    print(f"Restored from: {backup_dir}")
```

---

## 3. Vector Database Comparison

### Feature Comparison

| Feature | Chroma | Pinecone | pgvector | Qdrant | Milvus |
|---------|--------|----------|----------|--------|--------|
| **Deployment** | Embedded/Local | Cloud | Self-hosted | Both | Self-hosted |
| **License** | Apache 2.0 | Proprietary | PostgreSQL | Apache 2.0 | Apache 2.0 |
| **Start-up Time** | Seconds | Minutes | Minutes | Seconds | Minutes |
| **HNSW Support** | Yes | Yes | Yes | Yes | Yes |
| **Metadata Filters** | Yes | Yes | Yes | Yes | Yes |
| **Python Support** | Excellent | Excellent | Good | Good | Good |
| **LangChain Support** | Native | Native | Native | Native | Native |
| **Scaling** | Single machine | Cloud-native | Vertical/Horiz | Distributed | Distributed |
| **Cost** | Free | $70+/mo | Infrastructure | Free/$ | Infrastructure |

### Recommendations by Use Case

| Scenario | Recommended | Alternative |
|----------|-------------|-------------|
| Development/Prototyping | Chroma | - |
| Small Production (self-hosted) | pgvector | Chroma |
| Cloud-managed | Pinecone | Qdrant |
| Large-scale | Milvus | Qdrant |
| High performance with filters | Qdrant | pgvector |

---

## 4. LLM Options

### Cloud LLM Providers

| Provider | Model | Context | Strengths | Weaknesses |
|----------|-------|---------|-----------|-------------|
| **OpenAI** | GPT-4o | 128K | Best reasoning, medical benchmarks | Cost |
| **OpenAI** | GPT-4o-mini | 128K | Cost-effective | Less capable |
| **Anthropic** | Claude 3.5 Sonnet | 200K | Long context, safety | Slower |
| **Google** | Gemini 1.5 Pro | 2M | Long context, multimodal | Newer |

### Self-Hosted LLM Options

| Model | Size | Context | Requirements | Performance |
|-------|------|---------|--------------|-------------|
| **Llama 3.1** | 8B-70B | 8K-128K | 16GB+ RAM | Good |
| **Mistral** | 7B | 8K | 16GB RAM | Very Good |
| **Phi-3** | 3.8B | 4K | 8GB RAM | Good |
| **Qwen2** | 7B-72B | 32K+ | 32GB+ RAM | Excellent |

### Ollama Setup

```python
from langchain_ollama import ChatOllama
from langchain_openai import ChatOpenAI

# Option 1: Self-hosted with Ollama
llm = ChatOllama(
    model="llama3.1:70b",
    temperature=0,
    base_url="http://localhost:11434"
)

# Option 2: Cloud LLM
llm = ChatOpenAI(
    model="gpt-4o",
    temperature=0,
    max_tokens=2000
)
```

### Medical Accuracy Considerations

```python
# For medical use, consider lower temperature for consistency
llm_medical = ChatOpenAI(
    model="gpt-4o",
    temperature=0.1,  # Low temperature for factual accuracy
    max_tokens=2000,
    system_message="""You are a medical knowledge assistant. 
    Provide accurate, cited information based on medical literature.
    Always include source citations for factual claims."""
)
```

---

## 5. Embedding Models

### Model Comparison

| Model | Dimensions | Max Input | API Cost | Medical Accuracy |
|-------|------------|-----------|----------|------------------|
| text-embedding-3-large | 3072 | 8K | High | Best |
| text-embedding-3-small | 1536 | 8K | Low | Good |
| text-embedding-ada-002 | 1536 | 8K | Medium | Good |
| Cohere embed-v3.0 | 1024 | 512 | Medium | Good |
| nomic-embed-text | 768 | 8K | Free | Moderate |

### Implementation

```python
from langchain_openai import OpenAIEmbeddings
from langchain_ollama import OllamaEmbeddings

# OpenAI embeddings (recommended for production)
embeddings = OpenAIEmbeddings(
    model="text-embedding-3-large",
    dimensions=3072  # Optional: reduce to save storage
)

# Ollama embeddings (for self-hosted)
local_embeddings = OllamaEmbeddings(
    model="nomic-embed-text"
)

# Dimensionality reduction for storage optimization
embeddings_small = OpenAIEmbeddings(
    model="text-embedding-3-large",
    dimensions=1024  # Truncate to save ~66% storage
)
```

---

## 6. API Framework (FastAPI)

### Project Structure

```
medical-daemon/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI app entry point
│   ├── api/
│   │   ├── __init__.py
│   │   ├── routes/
│   │   │   ├── chat.py     # Chat endpoint
│   │   │   ├── search.py   # Search endpoint
│   │   │   └── drugs.py    # Drug lookup
│   │   └── deps.py         # Dependencies
│   ├── core/
│   │   ├── config.py       # Settings
│   │   └── security.py     # Auth
│   ├── services/
│   │   ├── llm.py          # LLM service
│   │   ├── vector.py       # Vector store
│   │   └── drugs.py       # Drug lookup
│   └── models/
│       ├── schemas.py      # Pydantic models
│       └── database.py     # DB models
├── data/
│   ├── chroma_db/          # Vector database
│   └── documents/          # Source documents
├── tests/
├── requirements.txt
└── docker-compose.yml
```

### FastAPI Application

```python
# app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager

from app.api.routes import chat, search, drugs
from app.core.config import settings
from app.services.vector import init_vector_store

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: Initialize vector store
    await init_vector_store()
    yield
    # Shutdown: Cleanup
    pass

app = FastAPI(
    title="Medical Knowledge Assistant",
    description="AI-powered medical information system",
    version="1.0.0",
    lifespan=lifespan
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Include routers
app.include_router(chat.router, prefix="/api/v1")
app.include_router(search.router, prefix="/api/v1")
app.include_router(drugs.router, prefix="/api/v1")

@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

### Chat Endpoint

```python
# app/api/routes/chat.py
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel
from typing import List, Optional
from app.services.llm import LLMService
from app.api.deps import get_llm_service

router = APIRouter()

class ChatMessage(BaseModel):
    message: str
    history: Optional[List[dict]] = None

class ChatResponse(BaseModel):
    answer: str
    sources: List[dict]
    citations: List[str]

@router.post("/chat", response_model=ChatResponse)
async def chat(
    request: ChatMessage,
    llm_service: LLMService = Depends(get_llm_service)
):
    try:
        result = await llm_service.process_message(
            message=request.message,
            history=request.history or []
        )
        return ChatResponse(**result)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

---

## 7. Database Infrastructure

### PostgreSQL (User Data)

```python
# app/models/database.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base
from sqlalchemy import Column, Integer, String, DateTime, Text, JSON
from datetime import datetime

DATABASE_URL = "postgresql://user:password@localhost:5432/medical_daemon"

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

class Conversation(Base):
    __tablename__ = "conversations"
    
    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(String, index=True)
    title = Column(String)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

class Message(Base):
    __tablename__ = "messages"
    
    id = Column(Integer, primary_key=True, index=True)
    conversation_id = Column(Integer, index=True)
    role = Column(String)  # "user" or "assistant"
    content = Column(Text)
    sources = Column(JSON, nullable=True)
    citations = Column(JSON, nullable=True)
    created_at = Column(DateTime, default=datetime.utcnow)

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### Redis (Caching)

```python
# app/services/cache.py
import redis
import json
from typing import Optional

redis_client = redis.Redis(
    host="localhost",
    port=6379,
    db=0,
    decode_responses=True
)

class CacheService:
    def __init__(self, ttl: int = 3600):
        self.ttl = ttl
    
    def get(self, key: str) -> Optional[dict]:
        data = redis_client.get(key)
        return json.loads(data) if data else None
    
    def set(self, key: str, value: dict):
        redis_client.setex(key, self.ttl, json.dumps(value))
    
    def delete(self, key: str):
        redis_client.delete(key)
    
    def get_or_set(self, key: str, factory):
        cached = self.get(key)
        if cached:
            return cached
        
        value = factory()
        self.set(key, value)
        return value
```

---

## 8. Task Queue (Celery)

```python
# app/tasks/ingestion.py
from celery import Celery
from celery.schedules import crontab

celery_app = Celery(
    "medical_daemon",
    broker="redis://localhost:6379/1",
    backend="redis://localhost:6379/2"
)

@celery_app.task
def index_documents(source: str, document_ids: list):
    """Background task to index documents."""
    # Import here to avoid circular imports
    from app.services.vector import VectorService
    from app.services.ingestion import DocumentIngester
    
    ingester = DocumentIngester()
    documents = ingester.fetch_documents(source, document_ids)
    
    vector_service = VectorService()
    vector_service.index_documents(documents)
    
    return {"indexed": len(documents)}

@celery_app.task
def sync_drug_database():
    """Periodic task to sync drug databases."""
    from app.services.drugs import DrugSyncService
    
    sync_service = DrugSyncService()
    sync_service.sync_npra()
    sync_service.sync_rxnorm()
    
    return {"status": "completed"}

# Schedule periodic tasks
celery_app.conf.beat_schedule = {
    "sync-drugs-daily": {
        "task": "app.tasks.ingestion.sync_drug_database",
        "schedule": crontab(hour=2, minute=0),  # 2 AM daily
    },
}
```

---

## 9. Docker Compose (Development)

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/medical_daemon
      - REDIS_URL=redis://redis:6379
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    depends_on:
      - db
      - redis
    volumes:
      - ./data:/app/data

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=medical_daemon
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

  chroma:
    image: chromadb/chroma:latest
    ports:
      - "8001:8000"
    volumes:
      - chroma_data:/chroma/chroma

volumes:
  postgres_data:
  redis_data:
  chroma_data:
```

---

## 10. Monitoring and Observability

### Logging Structure

```python
# app/core/logging.py
import logging
import sys
from logging.handlers import RotatingFileHandler
from app.core.config import settings

def setup_logging():
    logger = logging.getLogger("medical_daemon")
    logger.setLevel(logging.INFO)
    
    # Console handler
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setLevel(logging.INFO)
    
    # File handler
    file_handler = RotatingFileHandler(
        "logs/medical_daemon.log",
        maxBytes=10_000_000,  # 10MB
        backupCount=5
    )
    file_handler.setLevel(logging.DEBUG)
    
    # Format
    formatter = logging.Formatter(
        "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
    )
    console_handler.setFormatter(formatter)
    file_handler.setFormatter(formatter)
    
    logger.addHandler(console_handler)
    logger.addHandler(file_handler)
    
    return logger
```

### Metrics with Prometheus

```python
from prometheus_client import Counter, Histogram, generate_latest
from fastapi import Response

# Request metrics
REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "endpoint", "status"]
)

REQUEST_LATENCY = Histogram(
    "http_request_duration_seconds",
    "HTTP request latency",
    ["method", "endpoint"]
)

# RAG metrics
VECTOR_SEARCH_LATENCY = Histogram(
    "vector_search_duration_seconds",
    "Vector search latency"
)

LLM_CALL_LATENCY = Histogram(
    "llm_call_duration_seconds",
    "LLM call latency"
)

@router.get("/metrics")
async def metrics():
    return Response(content=generate_latest(), media_type="text/plain")
```

---

## 11. Environment Configuration

```python
# app/core/config.py
from pydantic_settings import BaseSettings
from functools import lru_cache
from typing import Optional

class Settings(BaseSettings):
    # API
    API_HOST: str = "0.0.0.0"
    API_PORT: int = 8000
    
    # Database
    DATABASE_URL: str = "postgresql://postgres:postgres@localhost:5432/medical_daemon"
    
    # Redis
    REDIS_URL: str = "redis://localhost:6379"
    
    # Vector Store
    CHROMA_PERSIST_DIR: str = "./data/chroma_db"
    VECTOR_DB_TYPE: str = "chroma"  # chroma, pinecone, pgvector
    
    # Pinecone (if using)
    PINECONE_API_KEY: Optional[str] = None
    PINECONE_INDEX: Optional[str] = "medical-knowledge"
    
    # LLM
    LLM_PROVIDER: str = "openai"  # openai, anthropic, ollama
    OPENAI_API_KEY: Optional[str] = None
    ANTHROPIC_API_KEY: Optional[str] = None
    OLLAMA_BASE_URL: str = "http://localhost:11434"
    OLLAMA_MODEL: str = "llama3.1:70b"
    
    # Embeddings
    EMBEDDING_MODEL: str = "text-embedding-3-large"
    EMBEDDING_DIMENSIONS: int = 3072
    
    # Medical Safety
    ENABLE_DISCLAIMERS: bool = True
    MAX_CITATIONS_PER_RESPONSE: int = 10
    
    # Rate Limiting
    RATE_LIMIT_PER_MINUTE: int = 30
    
    class Config:
        env_file = ".env"

@lru_cache()
def get_settings():
    return Settings()
```

---

## 12. Summary and Recommendations

### Development Environment

| Component | Technology |
|-----------|------------|
| Vector DB | Chroma (local) |
| LLM | OpenAI GPT-4o |
| Embeddings | text-embedding-3-large |
| Database | SQLite (for simplicity) |
| Cache | In-memory |

### Production Environment

| Component | Technology |
|-----------|------------|
| Vector DB | pgvector (self-hosted) or Pinecone |
| LLM | OpenAI GPT-4o or Ollama (self-hosted) |
| Embeddings | text-embedding-3-large |
| Database | PostgreSQL |
| Cache | Redis |
| Task Queue | Celery |
| Deployment | Docker + Kubernetes |

### Key Decisions

1. **Start with Chroma** - Easy to develop, can migrate later
2. **Use OpenAI initially** - Best accuracy, switch to Ollama if privacy/cost needed
3. **Plan for PostgreSQL** - User data will need relational storage
4. **Implement Redis caching** - Critical for performance with medical queries

---

## References

- Chroma: https://trychroma.com
- LangChain Chroma: https://python.langchain.com/docs/integrations/vectorstores/chroma
- FastAPI: https://fastapi.tiangolo.com
- LangChain: https://python.langchain.com
- Ollama: https://ollama.ai
