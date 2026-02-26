# RAG Pipeline Architecture

## Overview

The Retrieval-Augmented Generation (RAG) pipeline is the core of the medical knowledge assistant. It combines document retrieval with LLM generation to provide accurate, cited answers.

---

## Pipeline Stages

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Query     │───▶│   Retrieve  │───▶│  Generate   │───▶│   Format    │
│   Input     │    │   Context   │    │   Answer    │    │   Response  │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
      │                   │                   │                   │
      ▼                   ▼                   ▼                   ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Validate   │    │   Vector    │    │    LLM      │    │   Add       │
│  Input      │    │   Search    │    │   Prompt    │    │   Citations │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

---

## Stage 1: Query Input

### Input Validation
```python
from pydantic import BaseModel, Field
from typing import Optional

class QueryInput(BaseModel):
    message: str = Field(..., min_length=1, max_length=10000)
    conversation_id: Optional[str] = None
    return_sources: bool = True
    language: Optional[str] = "en"
    
    # Optional: filter retrieved sources
    source_types: Optional[list[str]] = ["pubmed", "guidelines"]
    min_evidence_level: Optional[int] = None  # 1-5
```

### Query Validation Rules
1. **Medical emergency detection** - Flag potential emergencies
2. **Query classification** - Determine intent (drug info, disease info, etc.)
3. **Language detection** - Handle multilingual queries

```python
class QueryValidator:
    EMERGENCY_KEYWORDS = [
        "chest pain", "difficulty breathing", "suicide",
        "overdose", "severe bleeding", "stroke symptoms"
    ]
    
    def validate(self, query: str) -> ValidationResult:
        # Check for emergencies
        if self._is_emergency(query):
            return ValidationResult(
                valid=False,
                is_emergency=True,
                message="Please seek immediate medical attention..."
            )
        
        # Check for valid query
        if len(query.strip()) < 3:
            return ValidationResult(
                valid=False,
                message="Query too short"
            )
        
        return ValidationResult(valid=True)
```

---

## Stage 2: Retrieve Context

### Query Enhancement
```python
class QueryEnhancer:
    """Improve query for better retrieval."""
    
    def __init__(self, llm):
        self.llm = llm
    
    def enhance(self, query: str, context: dict = None) -> str:
        """Rewrite query to improve retrieval."""
        
        prompt = f"""Given a user medical query, rewrite it to be more effective 
        for searching medical literature. Include relevant medical terms.
        
        Original: {query}
        
        Rewrite:"""
        
        enhanced = self.llm.invoke(prompt)
        return enhanced.content
```

### Vector Search
```python
class MedicalRetriever:
    """Retrieve relevant medical documents."""
    
    def __init__(self, vector_store, embedding_model):
        self.vector_store = vector_store
        self.embedding_model = embedding_model
        
        # Metadata filter for evidence levels
        self.evidence_weights = {
            "systematic_review": 1.0,
            "meta_analysis": 1.0,
            "rct": 0.9,
            "clinical_guideline": 0.85,
            "cohort_study": 0.7,
            "case_report": 0.5,
            "expert_opinion": 0.3
        }
    
    def retrieve(
        self,
        query: str,
        k: int = 5,
        source_filter: dict = None,
        min_evidence_level: int = None
    ) -> list[RetrievedDocument]:
        
        # Enhance query
        enhanced_query = self.query_enhancer.enhance(query)
        
        # Search vector store
        if source_filter:
            docs = self.vector_store.similarity_search(
                query=enhanced_query,
                k=k * 2,  # Get more, filter later
                filter=source_filter
            )
        else:
            docs = self.vector_store.similarity_search(
                query=enhanced_query,
                k=k * 2
            )
        
        # Filter by evidence level
        if min_evidence_level:
            docs = [
                d for d in docs 
                if self._get_evidence_level(d.metadata) <= min_evidence_level
            ]
        
        # Rank and limit
        docs = self._rank_by_evidence(docs)[:k]
        
        return docs
    
    def _rank_by_evidence(self, docs: list) -> list:
        """Re-rank documents by evidence level."""
        
        def evidence_score(doc):
            doc_type = doc.metadata.get("evidence_level", "expert_opinion")
            return self.evidence_weights.get(doc_type, 0.5)
        
        return sorted(docs, key=evidence_score, reverse=True)
```

---

## Stage 3: Generate Answer

### Prompt Template
```python
MEDICAL_SYSTEM_PROMPT = """You are a medical knowledge assistant. Your role is to 
provide accurate, evidence-based medical information using the provided context.

IMPORTANT GUIDELINES:
1. Always cite your sources using the format [Source: {title}, {year}]
2. If the provided context doesn't contain enough information, clearly state this
3. Distinguish between strong evidence (systematic reviews, RCTs) and weaker evidence
4. Never provide definitive diagnoses - suggest consulting healthcare providers
5. Include appropriate medical disclaimers

CONTEXT FROM MEDICAL LITERATURE:
{context}

USER QUESTION: {question}

ANSWER:"""

def build_prompt(query: str, retrieved_docs: list) -> str:
    """Build the prompt with context."""
    
    # Format context
    context_parts = []
    for i, doc in enumerate(retrieved_docs, 1):
        metadata = doc.metadata
        context_parts.append(
            f"""[Source {i}]: {metadata.get('title', 'Unknown')}
    Publication: {metadata.get('journal', 'Unknown')} ({metadata.get('year', 'N/A')})
    Evidence Level: {metadata.get('evidence_level', 'Unknown')}
    
    Content: {doc.page_content}"""
        )
    
    context = "\n\n".join(context_parts)
    
    return MEDICAL_SYSTEM_PROMPT.format(
        context=context,
        question=query
    )
```

### LLM Generation
```python
class MedicalGenerator:
    """Generate answers with medical context."""
    
    def __init__(self, llm, vector_store):
        self.llm = llm
        self.vector_store = vector_store
    
    async def generate(
        self,
        query: str,
        conversation_history: list = None,
        return_sources: bool = True
    ) -> GenerationResult:
        
        # Retrieve documents
        retrieved_docs = self.retriever.retrieve(query, k=5)
        
        # Build prompt
        prompt = build_prompt(query, retrieved_docs)
        
        # Add conversation history if available
        if conversation_history:
            prompt = self._add_history(prompt, conversation_history)
        
        # Generate
        response = await self.llm.ainvoke(prompt)
        
        # Extract citations
        citations = self._extract_citations(retrieved_docs, response.content)
        
        return GenerationResult(
            message=response.content,
            sources=retrieved_docs if return_sources else [],
            citations=citations,
            tokens_used=response.usage_metadata.get("total_tokens", 0) if hasattr(response, "usage_metadata") else 0
        )
```

---

## Stage 4: Format Response

### Response Model
```python
from pydantic import BaseModel
from typing import Optional, List
from datetime import datetime

class Source(BaseModel):
    title: str
    url: Optional[str] = None
    pmid: Optional[str] = None
    year: int
    journal: str
    evidence_level: str

class Citation(BaseModel):
    """A specific citation in the generated text."""
    text: str
    source_index: int
    start_pos: int
    end_pos: int

class GenerationResult(BaseModel):
    message: str
    sources: List[Source]
    citations: List[Citation]
    conversation_id: str
    model: str
    tokens_used: int
    created_at: datetime
    
    def to_response(self) -> dict:
        return {
            "message": self.message,
            "conversation_id": self.conversation_id,
            "sources": [s.model_dump() for s in self.sources],
            "citations": [c.model_dump() for c in self.citations],
            "model": self.model,
            "tokens_used": self.tokens_used
        }
```

### Citation Formatting
```python
class CitationFormatter:
    """Format citations in generated text."""
    
    def format(self, text: str, sources: list, citations: list) -> str:
        """Add citation markers to text."""
        
        formatted = text
        
        # Sort citations by position (reverse to preserve positions)
        sorted_citations = sorted(citations, key=lambda c: c.start_pos, reverse=True)
        
        for citation in sorted_citations:
            source = sources[citation.source_index - 1]
            marker = f"[{citation.source_index}]"
            
            # Insert marker after the referenced text
            formatted = (
                formatted[:citation.end_pos] + 
                marker + 
                formatted[citation.end_pos:]
            )
        
        return formatted
```

---

## Agentic RAG (Advanced)

For complex queries requiring multiple retrieval steps:

```python
from langchain.agents import create_agent
from langchain.tools import tool

@tool
def retrieve_medical_info(query: str) -> str:
    """Retrieve medical literature for a query."""
    docs = retriever.retrieve(query, k=3)
    return format_documents(docs)

@tool
def check_drug_interaction(drug1: str, drug2: str) -> str:
    """Check drug interactions using RxNorm."""
    return rxnorm_client.check_interaction(drug1, drug2)

@tool
def get_clinical_guideline(condition: str) -> str:
    """Get clinical guidelines for a condition."""
    docs = guidelines_retriever.retrieve(condition, k=2)
    return format_documents(docs)

# Create agent with tools
agent = create_agent(
    llm,
    tools=[retrieve_medical_info, check_drug_interaction, get_clinical_guideline],
    system_prompt="""You are a medical assistant. Use tools to find accurate, 
    evidence-based information. Always cite sources. If uncertain, say so."""
)

# Run agent
async def process_with_agent(query: str) -> GenerationResult:
    result = await agent.ainvoke({"messages": [HumanMessage(content=query)]})
    # Parse agent response and extract sources
    return parse_agent_result(result)
```

---

## Pipeline Flow Diagram

```
                                    ┌──────────────────────────────┐
                                    │         User Query           │
                                    │ "What are Metformin side    │
                                    │          effects?"           │
                                    └──────────────┬───────────────┘
                                                   │
                                    ┌──────────────▼───────────────┐
                                    │    1. Validate & Classify    │
                                    │  - Emergency check           │
                                    │  - Query classification      │
                                    └──────────────┬───────────────┘
                                                   │
                                    ┌──────────────▼───────────────┐
                                    │    2. Query Enhancement      │
                                    │  - Add medical terms         │
                                    │  - Expand abbreviations      │
                                    └──────────────┬───────────────┘
                                                   │
                                    ┌──────────────▼───────────────┐
                                    │    3. Retrieve Documents     │
                                    │  - Vector search (k=5)       │
                                    │  - Filter by evidence        │
                                    │  - Re-rank by quality        │
                                    └──────────────┬───────────────┘
                                                   │
                                    ┌──────────────▼───────────────┐
                                    │    4. Build Prompt           │
                                    │  - Add system instructions   │
                                    │  - Inject context            │
                                    │  - Add conversation history  │
                                    └──────────────┬───────────────┘
                                                   │
                                    ┌──────────────▼───────────────┐
                                    │    5. Generate Answer        │
                                    │  - Call LLM                  │
                                    │  - Extract citations         │
                                    │  - Add disclaimer            │
                                    └──────────────┬───────────────┘
                                                   │
                                    ┌──────────────▼───────────────┐
                                    │    6. Format Response        │
                                    │  - Add source list           │
                                    │  - Format citations          │
                                    │  - Return structured JSON    │
                                    └──────────────────────────────┘
```

---

## Configuration Options

```python
# Config for RAG pipeline
class RAGConfig:
    # Retrieval
    DEFAULT_K = 5
    MAX_K = 10
    SIMILARITY_THRESHOLD = 0.5
    
    # Evidence filtering
    DEFAULT_EVIDENCE_LEVEL = 3  # Allow up to cohort studies
    STRICT_EVIDENCE_LEVEL = 1   # Only systematic reviews
    
    # Generation
    MAX_TOKENS = 2000
    TEMPERATURE = 0.3  # Lower for factual medical info
    
    # Prompt
    INCLUDE_DISCLAIMER = True
    INCLUDE_EVIDENCE_LEVEL = True
```

---

## See Also

- [Research: RAG Implementation](../research/rag-implementation.md)
- [Research: Vector Databases](../research/vector-databases.md)
- [Data Sources: PubMed](../data-sources/pubmed.md)