# Medical Knowledge Assistant - Project Design

## 1. Project Overview

A fact-based medical chat system powered by LLM that provides accurate, cited medical information from authoritative sources. Designed for both consumers/patients and medical professionals, with web UI and API-first architecture for integration.

---

## 2. Core Architecture

### 2.1 Retrieval-Augmented Generation (RAG) Pattern

The foundation uses RAG - querying a vector database of medical literature rather than relying solely on LLM training data. This ensures:

- **Accuracy**: Grounded in real medical literature
- **Traceability**: Each answer can cite sources
- **Up-to-date**: Knowledge base can be refreshed

### 2.2 System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     API Layer (FastAPI)                     │
│  - REST endpoints for chat, search, source retrieval        │
│  - WebSocket for streaming responses                        │
├─────────────────────────────────────────────────────────────┤
│                  Chat Interface (Web UI)                    │
│  - React/Vue frontend                                       │
│  - Real-time streaming                                      │
│  - Source citation display                                  │
├─────────────────────────────────────────────────────────────┤
│                    LLM Orchestration                        │
│  - LangChain/LangGraph agent framework                      │
│  - Tool-based retrieval (PubMed, custom KB)                 │
│  - Citation generation & fact-checking                      │
├─────────────────────────────────────────────────────────────┤
│                   Vector Knowledge Base                     │
│  - Indexed medical literature (PubMed, guidelines)          │
│  - Chroma/Pinecone/PostgreSQL vector store                  │
├─────────────────────────────────────────────────────────────┤
│                   Data Sources (APIs)                       │
│  - PubMed/NCBI E-utilities                                  │
│  - Medical guidelines                                       │
│  - Drug databases (RxNorm, etc.)                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Technology Stack

### 3.1 LLM & Embedding Models

| Component | Options | Recommendation |
|-----------|---------|----------------|
| Primary LLM | OpenAI GPT-4, Claude, Gemini, local (Ollama) | Start with GPT-4 or Claude for quality, add Ollama for self-hosted |
| Embeddings | OpenAI text-embedding-3, Cohere, local | text-embedding-3-large for medical accuracy |

### 3.2 Framework & Orchestration

- **LangChain** or **LangGraph**: Agent orchestration, tool definitions
- **FastAPI**: REST API layer

### 3.3 Vector Database

- **Chroma** (local development)
- **Pinecone** (cloud)
- **PostgreSQL + pgvector** (self-hosted production)

### 3.4 Data Ingestion

- **PubMed E-utilities API**: Academic literature
- **Web loaders**: Medical guideline sites
- **PDF parsers**: Medical documents

---

## 4. Data Sources

### 4.1 Primary Medical Sources

| Source | Type | Access Method |
|--------|------|---------------|
| **PubMed** | Biomedical literature | NCBI E-utilities API |
| **PubMed Central** | Full-text articles | PubMed API / PMC |
| **MedlinePlus** | Consumer health info | Direct integration |
| **FDA** | Drug labels, recalls | Public API |
| **Clinical Guidelines** | Treatment guidelines | Scraped/API |
| **Drug Databases** | RxNorm, drug interactions | NIH APIs |

### 4.2 Source Prioritization

1. **Systematic reviews & meta-analyses** (highest evidence)
2. **Clinical trials & guidelines**
3. **Peer-reviewed research**
4. **Medical textbooks & references**
5. **Consumer health resources** (for patient-facing queries)

---

## 5. Key Features

### 5.1 Core Capabilities

- [ ] **Conversational Interface**: Multi-turn chat with medical context
- [ ] **Source Citation**: Every claim linked to source material
- [ ] **Evidence Level Indicators**: Distinguish between strong/weak evidence
- [ ] **Medical Vocabulary**: Understand and explain medical terminology
- [ ] **Query Enhancement**: Rewrite user questions for better retrieval

### 5.2 Safety Features

- [ ] **Disclaimer System**: Clear medical disclaimer on every response
- [ ] **Uncertainty Handling**: Flag when information is limited/conflicting
- [ ] **Emergency Detection**: Recognize and respond to medical emergencies
- [ ] **Confidence Scoring**: Show confidence level for each answer
- [ ] **Source Quality Filtering**: Prioritize high-quality sources

### 5.3 User Features

- [ ] **Search History**: Save and revisit conversations
- [ ] **Topic Filtering**: Filter by condition, drug, procedure
- [ ] **Export**: Export answers with citations (PDF, markdown)
- [ ] **Multi-language**: Support for multiple languages

---

## 6. API Design (MCP-Compatible)

The system can function as an LLM tool via Model Context Protocol:

```python
# Example: Using as a tool in another LLM application
@tool
def medical_search(query: str, sources: List[str] = ["pubmed"]) -> str:
    """Search medical literature and return cited answer."""
    ...

@tool  
def drug_interaction(drug1: str, drug2: str) -> Dict:
    """Check interactions between two drugs."""
    ...
```

### Endpoints

- `POST /chat` - Send message, get response with citations
- `GET /search` - Direct literature search
- `GET /sources/{id}` - Retrieve full source document
- `GET /drugs/{name}/interactions` - Drug interaction lookup
- `WS /chat/stream` - Streaming chat responses

---

## 7. Implementation Phases

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

## 8. Key Considerations

### 8.1 Medical AI Safety

- Never provide definitive medical diagnoses
- Always encourage consultation with healthcare providers
- Implement content filters for harmful medical advice
- Regular auditing of responses for accuracy

### 8.2 Cost Optimization

- Cache frequently asked questions
- Use smaller models for simple queries
- Implement retrieval caching

### 8.3 Quality Assurance

- Human review of sample responses
- Automated fact-checking against sources
- User feedback integration

---

## 9. Open Questions

1. **Data freshness**: How important is real-time medical literature vs. a curated static knowledge base?
2. **User authentication**: Do you need user accounts, or is anonymous acceptable?
3. **Regulatory considerations**: Are there specific compliance requirements (HIPAA, FDA, etc.)?
4. **Custom knowledge**: Should users be able to upload their own medical documents?

---

## 10. Research Notes

### RAG Implementation (from LangChain docs)

The core pattern involves:

1. **Indexing**: Load documents → Split into chunks → Embed → Store in vector DB
2. **Retrieval**: Given user query, retrieve relevant chunks from vector DB
3. **Generation**: Pass retrieved context + query to LLM for response

### PubMed/NCBI Access

- **E-utilities API**: `https://eutils.ncbi.nlm.nih.gov/entrez/eutils/`
- **PubMed Central**: Full-text articles available via PMC ID
- **Search capabilities**: MeSH terms, MeSH subheadings, field tags

### Key LangChain Components

- `create_agent`: Build RAG agents with tool-based retrieval
- `vector_store.similarity_search`: Retrieve relevant documents
- **Document loaders**: WebBaseLoader, PubMedLoader (community)
- **Text splitters**: RecursiveCharacterTextSplitter for chunking

### Embedding Model Options

- OpenAI text-embedding-3-large (recommended for medical)
- Cohere (good multilingual support)
- Ollama (local, privacy-focused)