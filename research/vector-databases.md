# Embedding Models & Vector Databases

## Embedding Model Options

| Model | Dimensions | Context | Best For |
|-------|------------|---------|----------|
| text-embedding-3-large | 3072 | 8192 | Medical accuracy (recommended) |
| text-embedding-3-small | 1536 | 8192 | Cost-effective |
| text-embedding-ada-002 | 1536 | 8192 | Legacy, widely tested |
| Cohere embed-english-v3.0 | 1024 | 512 | Multilingual support |
| Ollama (local) | varies | 8192 | Privacy, offline |

## Recommendations by Use Case

### For Medical Accuracy
```python
from langchain_openai import OpenAIEmbeddings

# Production - best accuracy
embeddings = OpenAIEmbeddings(
    model="text-embedding-3-large",
    dimensions=3072
)
```

### Cost-Effective Option
```python
# Development / prototyping
embeddings = OpenAIEmbeddings(
    model="text-embedding-3-small",
    dimensions=1536
)
```

### Local / Privacy-Focused
```python
from langchain_ollama import OllamaEmbeddings

embeddings = OllamaEmbeddings(
    model="nomic-embed-text"  # or "llama3", "mxbai-embed-large"
)
```

## Embedding Cost Optimization

1. **Use small model for prototyping**: 75% cost reduction
2. **Cache embeddings**: For static knowledge base
3. **Batch processing**: Process multiple documents together
4. **Dimensionality reduction**: If storage is concern (truncate to 1024)

```python
# Example: Caching embeddings
from langchain.embeddings import CacheBackedEmbeddings
from langchain.storage import LocalFileStore

store = LocalFileStore("./cache/")
cached_embeddings = CacheBackedEmbeddings.from_bytes_store(
    underlying_embeddings=embeddings,
    document_embedding_store=store,
    namespace=embeddings.model
)
```

## Vector Database Comparison

| Database | Type | Deployment | Best For |
|----------|------|------------|----------|
| **Chroma** | Local/Embedded | In-process | Development, prototyping |
| **Pinecone** | Cloud-hosted | Managed | Production cloud |
| **pgvector** | Self-hosted | Docker/VPS | Self-hosted production |
| **Qdrant** | Local/Cloud | Both | High performance, filters |
| **Milvus** | Distributed | Kubernetes | Large-scale |

## Chroma (Development)

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

# Initialize
vector_store = Chroma(
    collection_name="medical_knowledge",
    embedding_function=OpenAIEmbeddings(model="text-embedding-3-large"),
    persist_directory="./chroma_db"  # Local storage
)

# Add documents
from langchain_core.documents import Document

docs = [
    Document(
        page_content="Diabetes mellitus is a chronic metabolic disease...",
        metadata={"source": "pubmed", "pmid": "123456", "title": "Diabetes Overview"}
    )
]
vector_store.add_documents(documents=docs)

# Similarity search
results = vector_store.similarity_search(
    query="treatment for type 2 diabetes",
    k=3,
    filter={"source": "pubmed"}  # Optional filtering
)

# Similarity search with score
results_with_scores = vector_store.similarity_search_with_score(
    query="treatment for type 2 diabetes",
    k=3
)
```

## Pinecone (Cloud)

```python
from langchain_pinecone import PineconeVectorStore
from pinecone import Pinecone
from langchain_openai import OpenAIEmbeddings

# Setup
pc = Pinecone(api_key="your-api-key")
index = pc.Index("medical-knowledge")

vector_store = PineconeVectorStore(
    embedding=OpenAIEmbeddings(model="text-embedding-3-large"),
    index=index,
    text_key="text"
)

# Search with metadata filtering
results = vector_store.similarity_search(
    query="diabetes treatment",
    k=5,
    filter={
        "$and": [
            {"source": {"$eq": "pubmed"}},
            {"year": {"$gte": 2020}}
        ]
    }
)
```

## PostgreSQL + pgvector (Self-Hosted)

```python
from langchain_postgres import PGVector
from langchain_openai import OpenAIEmbeddings

# Connection string
connection = "postgresql+psycopg://user:pass@localhost:5432/medical_kb"

vector_store = PGVector(
    embeddings=OpenAIEmbeddings(model="text-embedding-3-large"),
    collection_name="medical_documents",
    connection=connection,
    pre_delete_collection=False
)
```

## Text Splitter Settings for Medical

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

# Medical-specific settings (smaller chunks for precision)
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,       # Smaller for medical precision
    chunk_overlap=200,     # Maintain context between chunks
    add_start_index=True,  # Track original document positions
    separators=[
        "\n\n\n",  # Major section breaks
        "\n\n",    # Section breaks
        "\n",      # Paragraph breaks
        ". ",      # Sentence breaks
        " ",       # Word breaks
        ""
    ]
)

# Split documents
chunks = text_splitter.split_documents(documents)

# Each chunk maintains metadata from original document
for chunk in chunks:
    print(f"Content: {chunk.page_content[:100]}...")
    print(f"Source: {chunk.metadata.get('source')}")
    print(f"PMID: {chunk.metadata.get('pmid')}")
```

## Metadata Schema for Medical Documents

```python
from typing import Optional
from pydantic import BaseModel

class MedicalDocMetadata(BaseModel):
    source: str                    # pubmed, fda, guidelines, etc.
    pmid: Optional[str] = None     # PubMed ID
    pmcid: Optional[str] = None    # PubMed Central ID
    title: str                     # Article title
    authors: Optional[str] = None  # Author list
    journal: Optional[str] = None  # Publication journal
    year: Optional[int] = None     # Publication year
    doi: Optional[str] = None      # Digital Object Identifier
    mesh_terms: Optional[list] = None  # MeSH headings
    evidence_level: Optional[str] = None  # systematic_review, rct, etc.
    url: Optional[str] = None      # Source URL