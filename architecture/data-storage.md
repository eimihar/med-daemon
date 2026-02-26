# Data Storage Architecture

## Overview

The system requires several types of data storage:
1. **Vector Store** - For embedding-based document retrieval
2. **Relational Store** - For structured data (conversations, users)
3. **Cache** - For performance optimization
4. **Object Storage** - For large files (PDFs, logs)

---

## 1. Vector Store

### Purpose
Store embedded medical documents for semantic search.

### Options Comparison

| Store | Type | Best For | Deployment |
|-------|------|----------|------------|
| **Chroma** | Local/Embedded | Development | In-process |
| **Pinecone** | Cloud-hosted | Production cloud | Managed |
| **pgvector** | Self-hosted | Self-hosted production | Docker/VPS |
| **Qdrant** | Both | Performance | Docker/Cloud |

### Recommended: pgvector (Self-Hosted)

```python
# Using PostgreSQL with pgvector extension
from langchain_postgres import PGVector
from langchain_postgres.engine import create_engine

# Create connection
engine = create_engine(
    "postgresql+psycopg://user:password@localhost:5432/medical_kb"
)

# Create vector store
vector_store = PGVector(
    embeddings=embeddings,
    collection_name="medical_documents",
    connection=engine,
    use_jsonb=True  # Store metadata as JSONB
)

# Document schema
# id: UUID (primary key)
# collection_id: UUID
# embedding: vector(1536)  # or 3072 for text-embedding-3-large
# document: text
# cmetadata: jsonb
# created_at: timestamp
```

### Collection Organization

```python
# Organize by source type
collections = {
    "pubmed_abstracts": "PubMed article abstracts",
    "pubmed_fulltext": "PubMed Central full text",
    "guidelines": "Clinical practice guidelines",
    "drug_labels": "FDA drug labels",
    "consumer_health": "MedlinePlus content"
}

# Each collection has different embedding settings
for name, description in collections.items():
    print(f"{name}: {description}")
```

---

## 2. Relational Store (PostgreSQL)

### Purpose
Store structured data: users, conversations, messages, settings.

### Schema Design

```sql
-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    metadata JSONB DEFAULT '{}'
);

-- Conversations table
CREATE TABLE conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    isarchived BOOLEAN DEFAULT FALSE,
    metadata JSONB DEFAULT '{}'
);

-- Messages table
CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID REFERENCES conversations(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL, -- 'user', 'assistant', 'system'
    content TEXT NOT NULL,
    sources JSONB, -- Array of source metadata
    model VARCHAR(100),
    tokens_used INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    metadata JSONB DEFAULT '{}'
);

-- Indexes
CREATE INDEX idx_conversations_user ON conversations(user_id);
CREATE INDEX idx_messages_conversation ON messages(conversation_id);
CREATE INDEX idx_messages_created ON messages(created_at DESC);
```

### Python with SQLAlchemy

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, relationship
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy import Column, String, Text, Boolean, Integer, ForeignKey, DateTime
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime
import uuid

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    email = Column(String(255), unique=True, nullable=False)
    name = Column(String(255))
    created_at = Column(DateTime, default=datetime.utcnow)
    is_active = Column(Boolean, default=True)
    metadata = Column(JSONB, default={})
    
    conversations = relationship("Conversation", back_populates="user")

class Conversation(Base):
    __tablename__ = "conversations"
    
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    user_id = Column(UUID(as_uuid=True), ForeignKey("users.id"))
    title = Column(String(500))
    created_at = Column(DateTime, default=datetime.utcnow)
    archived = Column(Boolean, default=False)
    metadata = Column(JSONB, default={})
    
    user = relationship("User", back_populates="conversations")
    messages = relationship("Message", back_populates="conversation", order_by="Message.created_at")

class Message(Base):
    __tablename__ = "messages"
    
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    conversation_id = Column(UUID(as_uuid=True), ForeignKey("conversations.id"))
    role = Column(String(20), nullable=False)
    content = Column(Text, nullable=False)
    sources = Column(JSONB)
    model = Column(String(100))
    tokens_used = Column(Integer)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    conversation = relationship("Conversation", back_populates="messages")

# Usage
engine = create_engine("postgresql://user:pass@localhost/medical_chat")
SessionLocal = sessionmaker(bind=engine)

def get_conversation_history(conversation_id: str, limit: int = 10) -> list:
    session = SessionLocal()
    messages = session.query(Message).filter(
        Message.conversation_id == conversation_id
    ).order_by(Message.created_at.desc()).limit(limit).all()
    session.close()
    return list(reversed(messages))
```

---

## 3. Cache Layer (Redis)

### Purpose
- Cache frequently asked queries
- Cache API responses
- Rate limiting
- Session storage

```python
import redis
import json
from typing import Optional

class CacheService:
    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis = redis.from_url(redis_url, decode_responses=True)
    
    # Query result cache
    async def get_cached_query(self, query: str) -> Optional[dict]:
        key = f"query:{hash(query)}"
        cached = self.redis.get(key)
        if cached:
            return json.loads(cached)
        return None
    
    async def cache_query(self, query: str, result: dict, ttl: int = 3600):
        key = f"query:{hash(query)}"
        self.redis.setex(key, ttl, json.dumps(result))
    
    # Rate limiting
    async def check_rate_limit(self, user_id: str, limit: int = 60, window: int = 60) -> bool:
        key = f"ratelimit:{user_id}"
        current = self.redis.get(key)
        
        if current and int(current) >= limit:
            return False
        
        pipe = self.redis.pipeline()
        pipe.incr(key)
        pipe.expire(key, window)
        pipe.execute()
        return True
    
    # Session cache
    async def set_session(self, session_id: str, data: dict, ttl: int = 86400):
        key = f"session:{session_id}"
        self.redis.setex(key, ttl, json.dumps(data))
    
    async def get_session(self, session_id: str) -> Optional[dict]:
        key = f"session:{session_id}"
        cached = self.redis.get(key)
        if cached:
            return json.loads(cached)
        return None
```

---

## 4. Object Storage

### Purpose
- Store source PDFs
- Store indexed documents
- Log storage

### Options

| Service | Use Case |
|---------|----------|
| **Local Disk** | Development |
| **S3/MinIO** | Production |
| **Azure Blob** | Azure deployment |

```python
import boto3
from botocore.config import Config

class ObjectStorage:
    def __init__(self, config: dict):
        self.client = boto3.client(
            's3',
            endpoint_url=config.get('endpoint'),
            aws_access_key_id=config.get('access_key'),
            aws_secret_access_key=config.get('secret_key'),
            config=Config(signature_version='s3v4')
        )
        self.bucket = config.get('bucket', 'medical-kb')
    
    async def upload_document(self, key: str, content: bytes, metadata: dict = None):
        kwargs = {"Body": content, "Bucket": self.bucket, "Key": key}
        if metadata:
            kwargs["Metadata"] = metadata
        self.client.put_object(**kwargs)
    
    async def download_document(self, key: str) -> bytes:
        response = self.client.get_object(Bucket=self.bucket, Key=key)
        return response['Body'].read()
    
    async def list_documents(self, prefix: str = "") -> list:
        response = self.client.list_objects_v2(Bucket=self.bucket, Prefix=prefix)
        return [obj['Key'] for obj in response.get('Contents', [])]
```

---

## 5. Data Flow Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                        APPLICATION LAYER                         │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  FastAPI    │  │  LangChain  │  │   Services  │             │
│  │  Server     │  │   Pipeline  │  │             │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
└─────────┼────────────────┼────────────────┼─────────────────────┘
          │                │                │
          ▼                ▼                ▼
┌──────────────────────────────────────────────────────────────────┐
│                      STORAGE LAYER                               │
│                                                                  │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐     │
│  │    PostgreSQL  │  │   Redis Cache  │  │   Vector DB    │     │
│  │                │  │                │  │   (pgvector)   │     │
│  │  - Users       │  │  - Query cache │  │                │     │
│  │  - Conversations   │  - Rate limits │  │  - Embeddings │     │
│  │  - Messages     │  │  - Sessions    │  │  - Documents  │     │
│  └────────────────┘  └────────────────┘  └────────────────┘     │
│                                                                  │
│  ┌────────────────┐  ┌────────────────┐                         │
│  │   S3/MinIO     │  │     Logs       │                         │
│  │                │  │   (Files/JSON) │                         │
│  │  - PDFs        │  │                │                         │
│  │  - Index data  │  │                │                         │
│  └────────────────┘  └────────────────┘                         │
└──────────────────────────────────────────────────────────────────┘
```

---

## 6. Development vs Production

### Development
```yaml
# docker-compose.dev.yml
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: medical_chat
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
  
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
  
  chroma:
    image: chromadb/chroma
    ports:
      - "8000:8000"
  
  app:
    build: .
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://dev:dev@postgres:5432/medical_chat
      REDIS_URL: redis://redis:6379
      VECTOR_STORE: chroma://chroma:8000

volumes:
  postgres_data:
```

### Production
```yaml
# docker-compose.prod.yml
services:
  app:
    build: .
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 2G
    environment:
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
      VECTOR_STORE: pinecone://${PINECONE_API_KEY}
  
  postgres:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data
    # Use managed PostgreSQL (RDS, Cloud SQL) in production
  
  redis:
    image: redis:7-alpine
    # Use managed Redis (ElastiCache, MemoryDB) in production
```

---

## 7. Backup & Recovery

```python
# Backup strategy
BACKUP_CONFIG = {
    "postgresql": {
        "frequency": "daily",
        "retention": "30 days",
        "method": "pg_dump to S3"
    },
    "redis": {
        "frequency": "hourly",
        "retention": "7 days",
        "method": "RDB snapshots"
    },
    "vector_store": {
        "frequency": "weekly",
        "retention": "90 days",
        "method": "export to S3"
    }
}
```

---

## See Also

- [Data Sources: PubMed](../data-sources/pubmed.md)
- [Research: Vector Databases](../research/vector-databases.md)