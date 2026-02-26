# Tech Stack Documentation

This folder contains detailed documentation for the technology stack used in the Medical Knowledge Assistant.

---

## Index

| Document | Description |
|----------|-------------|
| [overview.md](./overview.md) | Complete tech stack overview with Chroma DB details |

---

## Quick Reference

### Core Stack

| Layer | Technology |
|-------|------------|
| **API** | FastAPI |
| **RAG** | LangChain |
| **LLM** | OpenAI GPT-4o / Ollama |
| **Vector DB** | Chroma (dev) / pgvector (prod) |
| **Embeddings** | text-embedding-3-large |

### Development vs Production

| Component | Development | Production |
|-----------|-------------|------------|
| Vector DB | Chroma (local) | pgvector or Pinecone |
| LLM | OpenAI API | Ollama or OpenAI |
| Database | SQLite | PostgreSQL |
| Cache | In-memory | Redis |

---

## Key Documents

### Chroma DB
See [overview.md](./overview.md) for detailed Chroma documentation including:
- Installation and setup
- Basic operations (add, search, filter)
- Metadata filtering
- Collection management
- Production considerations

### Other Topics in overview.md
- LLM options (OpenAI, Claude, Ollama)
- Embedding models
- FastAPI application structure
- PostgreSQL and Redis setup
- Celery task queue
- Docker compose
- Monitoring with Prometheus
- Environment configuration

---

## See Also

- [Architecture Documents](../architecture/README.md)
- [Data Sources](../data-sources/README.md)
- [Research Documents](../research/README.md)
