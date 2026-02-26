# API Layer Options: FastAPI vs Express

This document compares two options for building the API layer: **Python/FastAPI** and **Node.js/Express**.

---

## Quick Comparison

| Aspect | FastAPI (Python) | Express (Node.js) |
|--------|------------------|-------------------|
| **Performance** | Excellent (ASGI) | Excellent |
| **Type Safety** | Pydantic (native) | TypeScript (optional) |
| **Learning Curve** | Low | Low |
| **LangChain Support** | Native | Via bindings |
| **Ecosystem** | Rich ML/AI libraries | Rich web ecosystem |
| **Real-time** | WebSocket native | WebSocket native |
| **Deployment** | Docker, ASGI servers | Docker, PM2 |
| **Team Expertise** | ML/Data science | Frontend/Backend |

---

## Option 1: FastAPI (Python)

### Pros

1. **Native LangChain Support**
   - LangChain is Python-first
   - All features available out of the box
   - No compatibility layer needed

2. **Type Safety with Pydantic**
   ```python
   from pydantic import BaseModel, Field
   from typing import Optional, List
   
   class ChatMessage(BaseModel):
       message: str = Field(..., min_length=1, max_length=10000)
       conversation_id: Optional[str] = None
       sources: bool = True
   
   class ChatResponse(BaseModel):
       message: str
       conversation_id: str
       sources: List[Source]
       citations: List[Citation]
   ```

3. **Automatic Documentation**
   - Swagger UI at `/docs`
   - ReDoc at `/redoc`
   - Auto-generated from type hints

4. **Async Native**
   ```python
   @app.post("/chat")
   async def chat(request: ChatMessage):
       result = await process_message(request)
       return result
   ```

5. **Rich ML/AI Ecosystem**
   - Direct access to all ML libraries
   - Easy integration with embeddings, vector DBs

### Cons

1. **Single Language** - Need Python for API + ML code
2. **GIL** - Not ideal for CPU-intensive parallel tasks (though async helps I/O)
3. **TypeScript** - Team may prefer TS for web stack

### Example Project Structure

```
med-daemon/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI app entry
│   ├── api/
│   │   ├── routes/
│   │   │   ├── chat.py
│   │   │   ├── search.py
│   │   │   └── tools.py
│   │   └── deps.py          # Dependencies
│   ├── services/
│   │   ├── chat_service.py
│   │   ├── search_service.py
│   │   └── tool_service.py
│   ├── models/
│   │   ├── request.py
│   │   └── response.py
│   └── core/
│       ├── config.py
│       └── security.py
├── tests/
├── requirements.txt
└── Dockerfile
```

### Example Code

```python
# app/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager

from app.api.routes import chat, search, tools
from app.core.config import settings

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: Initialize services
    await initialize_services()
    yield
    # Shutdown: Cleanup
    await cleanup_services()

app = FastAPI(
    title="Medical Knowledge API",
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

app.include_router(chat.router, prefix="/api/v1")
app.include_router(search.router, prefix="/api/v1")
app.include_router(tools.router, prefix="/api/v1")

@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

```python
# app/api/routes/chat.py
from fastapi import APIRouter, Depends, HTTPException
from app.models.request import ChatRequest
from app.models.response import ChatResponse
from app.services.chat_service import ChatService
from app.api.deps import get_chat_service

router = APIRouter()

@router.post("/chat", response_model=ChatResponse)
async def chat(
    request: ChatRequest,
    service: ChatService = Depends(get_chat_service)
):
    try:
        result = await service.process_message(
            message=request.message,
            conversation_id=request.conversation_id,
            return_sources=request.return_sources
        )
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.websocket("/chat/stream")
async def chat_stream(websocket, service: ChatService = Depends(get_chat_service)):
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_json()
            async for chunk in service.stream_message(data["message"]):
                await websocket.send_json(chunk)
    except Exception:
        await websocket.close()
```

---

## Option 2: Express (Node.js/TypeScript)

### Pros

1. **Full-Stack TypeScript**
   - Share types between frontend/backend
   - Consistent codebase for web team

2. **Mature Ecosystem**
   - Huge npm registry
   - Excellent middleware options

3. **Better for High-Concurrency**
   - Non-blocking I/O model
   - Great for many simultaneous connections

4. **Rich Web Frameworks**
   - tRPC for end-to-end types
   - Socket.io for WebSockets

### Cons

1. **LangChain.js** - Less mature than Python version
   ```typescript
   // LangChain.js - works but less documented
   import { ChatOpenAI } from "langchain/chat_models/openai";
   import { RetrievalQAChain } from "langchain/chains";
   ```

2. **Separate ML Pipeline**
   - Need Python service for heavy ML work
   - Or use LangChain.js (less capable)

3. **Type Safety** - More boilerplate needed

### Example Project Structure

```
med-daemon/
├── src/
│   ├── index.ts              # Express app entry
│   ├── routes/
│   │   ├── chat.ts
│   │   ├── search.ts
│   │   └── tools.ts
│   ├── services/
│   │   ├── chat.service.ts
│   │   ├── search.service.ts
│   │   └── tool.service.ts
│   ├── types/
│   │   ├── request.ts
│   │   └── response.ts
│   ├── middleware/
│   │   ├── auth.ts
│   │   └── validation.ts
│   └── config/
│       └── index.ts
├── tests/
├── package.json
└── Dockerfile
```

### Example Code

```typescript
// src/index.ts
import express from 'express';
import { createServer } from 'http';
import { WebSocketServer } from 'ws';
import cors from 'cors';
import chatRoutes from './routes/chat';
import searchRoutes from './routes/search';
import toolsRoutes from './routes/tools';
import { config } from './config';

const app = express();
const server = createServer(app);
const wss = new WebSocketServer({ server });

app.use(cors());
app.use(express.json());

app.use('/api/v1/chat', chatRoutes);
app.use('/api/v1/search', searchRoutes);
app.use('/api/v1/tools', toolsRoutes);

app.get('/health', (req, res) => {
  res.json({ status: 'healthy' });
});

// WebSocket for streaming
wss.on('connection', (ws) => {
  ws.on('message', async (message) => {
    const data = JSON.parse(message.toString());
    // Handle streaming chat
    const stream = await chatService.streamMessage(data.message);
    for await (const chunk of stream) {
      ws.send(JSON.stringify(chunk));
    }
  });
});

server.listen(config.port, () => {
  console.log(`Server running on port ${config.port}`);
});
```

```typescript
// src/routes/chat.ts
import { Router, Request, Response } from 'express';
import { ChatService } from '../services/chat.service';

const router = Router();
const chatService = new ChatService();

interface ChatRequest {
  message: string;
  conversation_id?: string;
  return_sources?: boolean;
}

router.post('/', async (req: Request, res: Response) => {
  try {
    const { message, conversation_id, return_sources } = req.body as ChatRequest;
    
    const result = await chatService.processMessage({
      message,
      conversationId: conversation_id,
      returnSources: return_sources ?? true
    });
    
    res.json(result);
  } catch (error) {
    res.status(500).json({ error: (error as Error).message });
  }
});

export default router;
```

---

## Hybrid Option: Node API + Python ML Service

Many production systems use this architecture:

```
┌──────────────┐     ┌──────────────┐
│   Node API   │────▶│  Python ML   │
│  (Express)   │◀────│   Service    │
└──────────────┘     └──────────────┘
     │                      │
     │              ┌───────┴───────┐
     │              │               │
     │         ┌────▼────┐    ┌─────▼─────┐
     │         │Vector DB│    │  LangChain│
     │         │         │    │  Pipeline │
     │         └─────────┘    └───────────┘
     │
     ▼
┌────────────┐
│  Clients   │
└────────────┘
```

### FastAPI ML Service Example

```python
# ml_service/main.py
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class QueryRequest(BaseModel):
    message: str
    conversation_id: str | None = None

class QueryResponse(BaseModel):
    message: str
    sources: list
    citations: list

@app.post("/query", response_model=QueryResponse)
async def process_query(request: QueryRequest):
    # RAG pipeline here
    result = await rag_pipeline.process(request.message)
    return result
```

### Node.js Calling Python Service

```typescript
// Call Python ML service
const response = await axios.post('http://ml-service:8000/query', {
  message: userMessage,
  conversation_id: conversationId
});

const result = response.data;
```

---

## Recommendation

| Scenario | Recommendation |
|----------|----------------|
| Small team, ML-focused | **FastAPI** - simpler, native LangChain |
| Full-stack TS team | **Express** - consistent stack |
| High traffic, real-time | **Express** - better concurrency |
| Need latest LLM features | **FastAPI** - LangChain is Python-first |
|已有 Python ML pipeline | **Express + Python service** |

### For This Project

Given that this is a medical LLM application requiring:
- Heavy use of LangChain for RAG
- Multiple external APIs (PubMed, RxNorm, FDA)
- Source citation requirements

**Recommendation**: Start with **FastAPI** because:
1. Native LangChain integration
2. All medical APIs have Python SDKs
3. Simpler single-language codebase
4. Easier to prototype RAG pipelines

You can always add a Node.js frontend or split services later if needed.

---

## See Also

- [FastAPI Official Docs](https://fastapi.tiangolo.com/)
- [Express Official Docs](https://expressjs.com/)
- [LangChain Python Docs](https://python.langchain.com/)
- [LangChain.js Docs](https://js.langchain.com/)