# RAG Implementation Research

## Overview

The core pattern involves:

1. **Indexing**: Load documents → Split into chunks → Embed → Store in vector DB
2. **Retrieval**: Given user query, retrieve relevant chunks from vector DB
3. **Generation**: Pass retrieved context + query to LLM for response

---

## Two RAG Approaches

| Approach | Description | Best For |
|----------|-------------|----------|
| **RAG Agent** | LLM uses tools to search when needed | Complex queries, multi-step reasoning |
| **RAG Chain** | Always检索, single LLM call | Simple queries, lower latency |

### RAG Agent (Tool-Based)

```python
from langchain.agents import create_agent
from langchain.tools import tool

@tool(response_format="content_and_artifact")
def retrieve_context(query: str):
    """Retrieve information from medical literature."""
    retrieved_docs = vector_store.similarity_search(query, k=3)
    serialized = "\n\n".join(
        f"Source: {doc.metadata}\nContent: {doc.page_content}"
        for doc in retrieved_docs
    )
    return serialized, retrieved_docs

agent = create_agent(
    model="claude-sonnet-4-5-20250929",
    tools=[retrieve_context],
    system_prompt="You are a medical assistant. Use the tool to retrieve relevant medical literature."
)

# Use the agent
for event in agent.stream(
    {"messages": [{"role": "user", "content": "What are the treatments for type 2 diabetes?"}]},
    stream_mode="values"
):
    event["messages"][-1].pretty_print()
```

### RAG Chain (Two-Step)

```python
from langchain.agents.middleware import AgentMiddleware, AgentState
from typing import Any
from langchain_core.documents import Document

class RetrieveDocumentsMiddleware(AgentMiddleware[AgentState]):
    def before_model(self, state: AgentState) -> dict[str, Any] | None:
        last_message = state["messages"][-1]
        retrieved_docs = vector_store.similarity_search(last_message.text, k=3)
        
        docs_content = "\n\n".join(doc.page_content for doc in retrieved_docs)
        
        system_message = (
            "Use the following medical literature to answer the question:\n\n"
            f"{docs_content}"
        )
        
        return {
            "messages": [last_message.model_copy(
                update={"content": f"{last_message.text}\n\n{system_message}"}
            )]
        }

agent = create_agent(model, tools=[], middleware=[RetrieveDocumentsMiddleware()])
```

---

## Comparison

| Criteria | RAG Agent | RAG Chain |
|----------|-----------|-----------|
| **Latency** | Higher (multiple calls) | Lower (single call) |
| **Flexibility** | Can decide when to search | Always searches |
| **Complex queries** | Better | Limited |
| **Simple queries** | Overkill | Ideal |
| **Cost** | Higher | Lower |

---

## Recommendation for Medical Chat

Use **RAG Agent** for medical queries because:

1. **Complex medical questions** often require multiple retrieval steps
2. **Accuracy is critical** - agent can self-verify by searching multiple sources
3. **Medical literature varies** - some queries need specific studies, others general info
4. **Multi-step reasoning** - can first search, then refine based on results