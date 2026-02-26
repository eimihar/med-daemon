# Architecture Documentation

This folder contains detailed architecture documentation for the Medical Knowledge Assistant.

---

## Index

| Document | Description |
|----------|-------------|
| [system-overview.md](./system-overview.md) | High-level system architecture |
| [api-layer-options.md](./api-layer-options.md) | FastAPI vs Express comparison |
| [rag-pipeline.md](./rag-pipeline.md) | RAG pipeline implementation |
| [data-storage.md](./data-storage.md) | Storage architecture |

---

## Quick Summary

### Recommended Stack

| Layer | Technology |
|-------|------------|
| **API** | FastAPI (Python) |
| **RAG Framework** | LangChain |
| **LLM** | OpenAI GPT-4 (start), Ollama (self-hosted) |
| **Vector DB** | Chroma (dev), pgvector (self-hosted), Pinecone (cloud) |
| **Relational DB** | PostgreSQL |
| **Cache** | Redis |

### Why FastAPI over Express?

1. **Native LangChain support** - All features available
2. **Python ecosystem** - Easy ML/AI library integration
3. **Simpler codebase** - Single language throughout
4. **Auto-generated docs** - Swagger UI out of box

---

## Architecture Highlights

### RAG Pipeline
- Query validation & enhancement
- Vector search with evidence-level ranking
- Context-aware prompt building
- Source citation extraction

### Data Storage
- **Vector**: Medical document embeddings
- **Relational**: Users, conversations, messages
- **Cache**: Query results, rate limiting
- **Object**: PDFs, indexed documents

---

## See Also

- [Research Documents](../research/)
- [Data Sources](../data-sources/)
- [Tech Stack](./tech-stack/)