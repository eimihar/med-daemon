# Research Documentation

This folder contains research notes, findings, and technical investigations for the Medical Knowledge Assistant.

---

## Index

| Document | Description |
|----------|-------------|
| [rag-implementation.md](./rag-implementation.md) | RAG agent vs chain comparison |
| [pubmed-api.md](./pubmed-api.md) | PubMed E-utilities API documentation |
| [vector-databases.md](./vector-databases.md) | Embedding models & vector DB options |
| [medical-apis.md](./medical-apis.md) | RxNorm, FDA, MedlinePlus APIs |

---

## Quick Summary

### Key Research Areas

1. **RAG Implementation**: Compared agent vs chain patterns for medical QA
2. **Vector Databases**: Evaluated Chroma, Pinecone, pgvector, Qdrant, Milvus
3. **PubMed API**: Documented E-utilities for literature retrieval
4. **Medical APIs**: Coverage of drug databases and health information sources

### Recommendations

- Use LangChain for RAG implementation
- Start with Chroma for development, migrate to pgvector/Pinecone for production
- Leverage PubMed E-utilities for academic literature
- Integrate RxNorm and FDA for drug information

---

## See Also

- [Architecture Documents](../architecture/)
- [Data Sources](../data-sources/)
- [Tech Stack](../tech-stack/)
