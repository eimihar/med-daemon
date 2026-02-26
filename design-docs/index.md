# Medical Knowledge Assistant - Project Design

A fact-based medical chat system powered by LLM that provides accurate, cited medical information from authoritative sources.

---

## Index

- [Architecture](./architecture/README.md) - System architecture documents
- [Tech Stack](./tech-stack/README.md) - Technology choices and comparisons
- [Data Sources](./data-sources/README.md) - Medical data sources documentation
- [Features](./features/README.md) - Feature specifications
- [API Design](./api-design/README.md) - API endpoints and design
- [Implementation](./implementation/README.md) - Implementation phases
- [Safety](./safety/README.md) - Medical safety features
- [Considerations](./considerations/README.md) - Key considerations
- [Research](./research/README.md) - Research notes and findings

---

## Quick Reference

### Core Architecture
- **Pattern**: Retrieval-Augmented Generation (RAG)
- **Framework**: LangChain / LangGraph
- **API Layer**: FastAPI

### Technology Stack
- **LLM**: OpenAI GPT-4 / Claude (start), Ollama (self-hosted)
- **Embeddings**: text-embedding-3-large
- **Vector DB**: Chroma (dev), Pinecone (cloud), pgvector (self-hosted)
- **Details**: See [Tech Stack](./tech-stack/README.md) documentation

### Data Sources
- **PubMed**: Academic literature
- **RxNorm**: Drug database
- **FDA**: Drug labels
- **MedlinePlus**: Consumer health

### Key Features
- Source citation for every claim
- Evidence level indicators
- Drug interaction checking
- Medical disclaimer system

---

## Research Documents

See [`research/`](./research/README.md) folder for detailed research:

1. **rag-implementation.md** - RAG agent vs chain comparison
2. **pubmed-api.md** - PubMed E-utilities API documentation
3. **vector-databases.md** - Embedding models & vector DB options
4. **medical-apis.md** - RxNorm, FDA, MedlinePlus APIs

---

## Implementation Phases

### Phase 1: Foundation (MVP)
- [ ] Basic RAG pipeline with PubMed
- [ ] Simple chat interface
- [ ] Source citation display
- [ ] Basic disclaimer

### Phase 2: Enhanced Accuracy
- [ ] Multi-source retrieval
- [ ] Citation verification
- [ ] Evidence level scoring
- [ ] Drug database integration

### Phase 3: Production Ready
- [ ] User authentication
- [ ] Rate limiting
- [ ] Monitoring & logging
- [ ] Self-hosted deployment

### Phase 4: Advanced Features
- [ ] Multi-language support
- [ ] Voice interface
- [ ] Integration plugins
- [ ] Custom knowledge base uploads

---

## Open Questions

1. **Data freshness**: Real-time literature vs curated static knowledge base?
2. **User authentication**: User accounts or anonymous acceptable?
3. **Regulatory**: HIPAA, FDA compliance needed?
4. **Custom knowledge**: Allow user document uploads?

---

*Last updated: 2026-02-26*