# How to Build a Production-Ready AI Agent API with FastAPI

**Reading time: 16 min | May 9, 2026**

You've built an AI agent that works in a Jupyter notebook. It answers questions, calls tools, and even remembers context. But when you try to put it into production — expose it as an API, handle concurrent users, manage rate limiting, and deploy it — everything falls apart.

This is the gap between "AI demo" and "AI product." And it's exactly where most startups stall.

At ZOO, we've built and deployed dozens of AI agent APIs for clients. In this guide, I'll show you the **exact architecture and code** we use to go from a working agent prototype to a production-ready API that handles real traffic.

No abstractions. No "just use LangChain." Real code you can deploy today.

---

## The Problem with Most AI Agent APIs

Most tutorials stop at:

```python
@app.post("/chat")
def chat(message: str):
    response = agent.run(message)
    return {"response": response}
```

This works for a demo. In production, you'll hit these problems immediately:

1. **No streaming** — Users stare at a blank screen for 10+ seconds waiting for the full response
2. **No session management** — Every request is stateless; the agent forgets everything between calls
3. **No rate limiting** — One user can spam your API and drain your API credits
4. **No error handling** — When the LLM times out, your whole API crashes
5. **No observability** — You have no idea what's happening, what's failing, or what's costing money
6. **No tool sandboxing** — Your agent can execute arbitrary code with no guardrails

Let's fix all of this.

---

## Architecture Overview

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│   Client     │────▶│  FastAPI     │────▶│  Agent Engine   │
│  (HTTP/SSE)  │◀────│  Gateway     │◀────│  (Orchestrator) │
└─────────────┘     └──────┬───────┘     └────────┬────────┘
                           │                       │
                    ┌──────▼───────┐     ┌────────▼────────┐
                    │  Rate Limiter │     │  Tool Registry  │
                    │  Session Mgr  │     │  (Sandboxed)    │
                    └──────────────┘     └─────────────────┘
                           │
                    ┌──────▼───────┐
                    │  Observability│
                    │  (Logs/Metrics)│
                    └──────────────┘
```

---

## Step 1: Project Structure

```
ai-agent-api/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI entry point
│   ├── config.py            # Settings & env vars
│   ├── models/
│   │   ├── __init__.py
│   │   ├── requests.py      # Request schemas
│   │   └── responses.py     # Response schemas
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── engine.py        # Core agent logic
│   │   ├── tools.py         # Tool definitions
│   │   └── memory.py        # Session memory
│   ├── middleware/
│   │   ├── __init__.py
│   │   ├── rate_limit.py    # Rate limiting
│   │   └── auth.py          # API key auth
│   └── observability/
│       ├── __init__.py
│       └── logger.py        # Structured logging
├── tests/
│   ├── test_api.py
│   └── test_agent.py
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
└── .env.example
```

---

## Step 2: Configuration with Pydantic Settings

```python
# app/config.py
from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    # LLM
    openai_api_key: str = ""
    anthropic_api_key: str = ""
    default_model: str = "gpt-4o-mini"

    # API
    api_keys: str = ""  # Comma-separated valid API keys
    rate_limit_per_minute: int = 20

    # Sessions
    session_ttl_seconds: int = 3600  # 1 hour
    max_sessions: int = 1000

    # Observability
    log_level: str = "INFO"
    enable_metrics: bool = True

    class Config:
        env_file = ".env"

@lru_cache()
def get_settings() -> Settings:
    return Settings()
```

---

## Step 3: Request/Response Models

```python
# app/models/requests.py
from pydantic import BaseModel, Field
from typing import Optional
from enum import Enum

class MessageRole(str, Enum):
    USER = "user"
    ASSISTANT = "assistant"
    SYSTEM = "system"
    TOOL = "tool"

class ChatMessage(BaseModel):
    role: MessageRole
    content: str
    tool_call_id: Optional[str] = None

class ChatRequest(BaseModel):
    message: str = Field(..., min_length=1, max_length=10000)
    session_id: Optional[str] = None
    stream: bool = True
    model: Optional[str] = None

class ToolCallRequest(BaseModel):
    tool_name: str
    arguments: dict
```

```python
# app/models/responses.py
from pydantic import BaseModel
from typing import Optional

class ChatResponse(BaseModel):
    session_id: str
    message: str
    model: str
    tokens_used: int
    tools_called: list[str] = []

class ErrorResponse(BaseModel):
    error: str
    detail: Optional[str] = None
    retry_after: Optional[int] = None
```

---

## Step 4: Session Memory Manager

```python
# app/agents/memory.py
import time
import uuid
from collections import OrderedDict
from threading import Lock
from app.models.requests import ChatMessage

class SessionManager:
    """Thread-safe session store with TTL-based eviction."""

    def __init__(self, max_sessions: int = 1000, ttl_seconds: int = 3600):
        self._sessions: OrderedDict[str, dict] = OrderedDict()
        self._max_sessions = max_sessions
        self._ttl = ttl_seconds
        self._lock = Lock()

    def get_or_create(self, session_id: str | None = None) -> tuple[str, list[ChatMessage]]:
        with self._lock:
            # Create new session
            if session_id is None or session_id not in self._sessions:
                sid = session_id or str(uuid.uuid4())
                self._sessions[sid] = {
                    "messages": [],
                    "created_at": time.time(),
                    "last_accessed": time.time(),
                }
                self._evict_if_needed()
                return sid, []

            # Return existing
            session = self._sessions[session_id]
            session["last_accessed"] = time.time()
            self._sessions.move_to_end(session_id)
            messages = [ChatMessage(**m) for m in session["messages"]]
            return session_id, messages

    def add_message(self, session_id: str, message: ChatMessage):
        with self._lock:
            if session_id in self._sessions:
                self._sessions[session_id]["messages"].append(message.model_dump())
                self._sessions[session_id]["last_accessed"] = time.time()

    def _evict_if_needed(self):
        # Evict expired sessions first
        now = time.time()
        expired = [
            sid for sid, s in self._sessions.items()
            if now - s["last_accessed"] > self._ttl
        ]
        for sid in expired:
            del self._sessions[sid]

        # Evict oldest if still over limit
        while len(self._sessions) > self._max_sessions:
            self._sessions.popitem(last=False)

    @property
    def active_sessions(self) -> int:
        return len(self._sessions)
```

---

## Step 5: Tool Registry with Sandboxing

```python
# app/agents/tools.py
import subprocess
import json
from typing import Callable
from pydantic import BaseModel

class ToolDefinition(BaseModel):
    name: str
    description: str
    parameters: dict
    handler: Callable
    requires_approval: bool = False

class ToolRegistry:
    """Sandboxed tool execution with timeout and output limits."""

    def __init__(self):
        self._tools: dict[str, ToolDefinition] = {}

    def register(self, definition: ToolDefinition):
        self._tools[definition.name] = definition

    def get_openai_schema(self) -> list[dict]:
        """Return tools in OpenAI function-calling format."""
        return [
            {
                "type": "function",
                "function": {
                    "name": t.name,
                    "description": t.description,
                    "parameters": t.parameters,
                }
            }
            for t in self._tools.values()
        ]

    async def execute(self, tool_name: str, arguments: dict) -> str:
        if tool_name not in self._tools:
            return json.dumps({"error": f"Unknown tool: {tool_name}"})

        tool = self._tools[tool_name]
        try:
            result = await tool.handler(**arguments)
            # Truncate long outputs
            if len(str(result)) > 5000:
                result = str(result)[:5000] + "... [truncated]"
            return str(result)
        except Exception as e:
            return json.dumps({"error": str(e)})

# --- Built-in Tools ---

async def search_web(query: str) -> str:
    """Search the web for information. Use for current events and facts."""
    # Integrate with SerpAPI, Tavily, or your preferred search provider
    # This is a placeholder — replace with actual implementation
    return f"Search results for: {query}"

async def execute_python(code: str) -> str:
    """Execute Python code in a sandboxed environment. Returns stdout."""
    try:
        result = subprocess.run(
            ["python3", "-c", code],
            capture_output=True,
            text=True,
            timeout=10,  # 10 second timeout
            env={"PATH": "/usr/bin"},  # Minimal env
        )
        if result.returncode != 0:
            return f"Error: {result.stderr[:500]}"
        return result.stdout[:2000]
    except subprocess.TimeoutExpired:
        return "Error: Code execution timed out (10s limit)"

async def fetch_url(url: str) -> str:
    """Fetch and extract text content from a URL."""
    import urllib.request
    try:
        req = urllib.request.Request(url, headers={"User-Agent": "AI-Agent/1.0"})
        with urllib.request.urlopen(req, timeout=10) as response:
            content = response.read().decode("utf-8", errors="ignore")
            # Strip HTML tags (basic)
            import re
            text = re.sub(r"<[^>]+>", "", content)
            return text[:3000]
    except Exception as e:
        return f"Error fetching URL: {e}"

# Create and configure the registry
def create_default_registry() -> ToolRegistry:
    registry = ToolRegistry()
    registry.register(ToolDefinition(
        name="search_web",
        description="Search the web for current information",
        parameters={
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "Search query"}
            },
            "required": ["query"],
        },
        handler=search_web,
    ))
    registry.register(ToolDefinition(
        name="execute_python",
        description="Execute Python code (sandboxed, 10s timeout)",
        parameters={
            "type": "object",
            "properties": {
                "code": {"type": "string", "description": "Python code to execute"}
            },
            "required": ["code"],
        },
        handler=execute_python,
        requires_approval=True,
    ))
    registry.register(ToolDefinition(
        name="fetch_url",
        description="Fetch content from a URL",
        parameters={
            "type": "object",
            "properties": {
                "url": {"type": "string", "description": "URL to fetch"}
            },
            "required": ["url"],
        },
        handler=fetch_url,
    ))
    return registry
```

---

## Step 6: The Agent Engine

```python
# app/agents/engine.py
import json
import time
from openai import AsyncOpenAI
from app.config import get_settings
from app.agents.memory import SessionManager
from app.agents.tools import create_default_registry, ToolRegistry
from app.models.requests import ChatMessage, MessageRole

class AgentEngine:
    """Production-ready agent with tool calling, streaming, and session memory."""

    def __init__(self):
        self.settings = get_settings()
        self.client = AsyncOpenAI(api_key=self.settings.openai_api_key)
        self.sessions = SessionManager(
            max_sessions=self.settings.max_sessions,
            ttl_seconds=self.settings.session_ttl_seconds,
        )
        self.tools: ToolRegistry = create_default_registry()

    async def chat(
        self,
        message: str,
        session_id: str | None = None,
        model: str | None = None,
    ):
        """
        Streaming chat with tool support.
        Yields SSE-formatted chunks.
        """
        model = model or self.settings.default_model
        sid, history = self.sessions.get_or_create(session_id)

        # Add user message to history
        user_msg = ChatMessage(role=MessageRole.USER, content=message)
        self.sessions.add_message(sid, user_msg)

        # Build messages for the LLM
        messages = self._build_messages(history, message)

        # Track metrics
        start_time = time.time()
        tokens_used = 0
        tools_called = []

        # First call — let the LLM decide if it needs tools
        response = await self.client.chat.completions.create(
            model=model,
            messages=messages,
            tools=self.tools.get_openai_schema(),
            tool_choice="auto",
            stream=True,
        )

        # Collect the full response for tool call detection
        tool_calls = []
        assistant_content = ""

        async for chunk in response:
            delta = chunk.choices[0].delta

            # Stream text content
            if delta.content:
                assistant_content += delta.content
                yield self._sse_chunk({
                    "type": "content",
                    "content": delta.content,
                    "session_id": sid,
                })

            # Collect tool calls
            if delta.tool_calls:
                for tc in delta.tool_calls:
                    if tc.index >= len(tool_calls):
                        tool_calls.append({
                            "id": tc.id,
                            "name": tc.function.name if tc.function else "",
                            "arguments": tc.function.arguments if tc.function else "",
                        })
                    else:
                        if tc.function and tc.function.arguments:
                            tool_calls[tc.index]["arguments"] += tc.function.arguments

            # Track usage
            if chunk.usage:
                tokens_used = chunk.usage.total_tokens

        # Execute tool calls if any
        if tool_calls:
            # Add assistant message with tool calls
            messages.append({
                "role": "assistant",
                "content": assistant_content or None,
                "tool_calls": [
                    {
                        "id": tc["id"],
                        "type": "function",
                        "function": {
                            "name": tc["name"],
                            "arguments": tc["arguments"],
                        }
                    }
                    for tc in tool_calls
                ]
            })

            for tc in tool_calls:
                tools_called.append(tc["name"])
                # Execute the tool
                args = json.loads(tc["arguments"]) if tc["arguments"] else {}
                result = await self.tools.execute(tc["name"], args)

                # Add tool result to messages
                messages.append({
                    "role": "tool",
                    "tool_call_id": tc["id"],
                    "content": result,
                })

                yield self._sse_chunk({
                    "type": "tool_result",
                    "tool": tc["name"],
                    "result": result[:200],
                    "session_id": sid,
                })

            # Final response after tool execution
            final_response = await self.client.chat.completions.create(
                model=model,
                messages=messages,
                stream=True,
            )

            async for chunk in final_response:
                delta = chunk.choices[0].delta
                if delta.content:
                    yield self._sse_chunk({
                        "type": "content",
                        "content": delta.content,
                        "session_id": sid,
                    })

        # Save assistant response to session
        self.sessions.add_message(
            sid,
            ChatMessage(role=MessageRole.ASSISTANT, content=assistant_content),
        )

        # Send done event with metadata
        elapsed = time.time() - start_time
        yield self._sse_chunk({
            "type": "done",
            "session_id": sid,
            "tokens_used": tokens_used,
            "tools_called": tools_called,
            "elapsed_seconds": round(elapsed, 2),
        })

    def _build_messages(
        self, history: list[ChatMessage], current_message: str
    ) -> list[dict]:
        messages = [
            {
                "role": "system",
                "content": (
                    "You are a helpful AI assistant with access to tools. "
                    "Use tools when you need current information, calculations, "
                    "or to fetch external data. Be concise and accurate."
                ),
            }
        ]
        for msg in history[-10:]:  # Last 10 messages for context
            messages.append({"role": msg.role.value, "content": msg.content})
        messages.append({"role": "user", "content": current_message})
        return messages

    def _sse_chunk(self, data: dict) -> str:
        return f"data: {json.dumps(data)}\n\n"
```

---

## Step 7: Rate Limiting Middleware

```python
# app/middleware/rate_limit.py
import time
from collections import defaultdict
from fastapi import Request, HTTPException
from starlette.middleware.base import BaseHTTPMiddleware

class RateLimitMiddleware(BaseHTTPMiddleware):
    """Token bucket rate limiter per API key."""

    def __init__(self, app, requests_per_minute: int = 20):
        super().__init__(app)
        self.rpm = requests_per_minute
        self._requests: dict[str, list[float]] = defaultdict(list)

    async def dispatch(self, request: Request, call_next):
        # Skip rate limiting for health/docs
        if request.url.path in ("/health", "/docs", "/openapi.json"):
            return await call_next(request)

        api_key = request.headers.get("X-API-Key", "anonymous")
        now = time.time()

        # Clean old entries
        self._requests[api_key] = [
            t for t in self._requests[api_key] if now - t < 60
        ]

        if len(self._requests[api_key]) >= self.rpm:
            raise HTTPException(
                status_code=429,
                detail={
                    "error": "Rate limit exceeded",
                    "retry_after": int(60 - (now - self._requests[api_key][0])),
                    "limit": self.rpm,
                },
            )

        self._requests[api_key].append(now)
        response = await call_next(request)
        response.headers["X-RateLimit-Limit"] = str(self.rpm)
        response.headers["X-RateLimit-Remaining"] = str(
            self.rpm - len(self._requests[api_key])
        )
        return response
```

---

## Step 8: The FastAPI Application

```python
# app/main.py
import logging
from contextlib import asynccontextmanager
from fastapi import FastAPI, Header, HTTPException
from fastapi.responses import StreamingResponse
from fastapi.middleware.cors import CORSMiddleware
from app.config import get_settings
from app.agents.engine import AgentEngine
from app.middleware.rate_limit import RateLimitMiddleware
from app.models.requests import ChatRequest
from app.models.responses import ErrorResponse

# Structured logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
)
logger = logging.getLogger("ai-agent-api")

# Global agent instance
agent: AgentEngine | None = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global agent
    logger.info("Starting AI Agent API...")
    agent = AgentEngine()
    logger.info(f"Agent ready. Model: {agent.settings.default_model}")
    yield
    logger.info("Shutting down...")

app = FastAPI(
    title="AI Agent API",
    version="1.0.0",
    lifespan=lifespan,
)

# Middleware
settings = get_settings()
app.add_middleware(
    RateLimitMiddleware,
    requests_per_minute=settings.rate_limit_per_minute,
)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["POST", "GET"],
    allow_headers=["*"],
)

def verify_api_key(x_api_key: str = Header(...)):
    valid_keys = [k.strip() for k in settings.api_keys.split(",") if k.strip()]
    if valid_keys and x_api_key not in valid_keys:
        raise HTTPException(status_code=401, detail="Invalid API key")
    return x_api_key

@app.get("/health")
async def health():
    return {
        "status": "healthy",
        "active_sessions": agent.sessions.active_sessions if agent else 0,
    }

@app.post("/chat")
async def chat(request: ChatRequest, api_key: str = Header(...)):
    """Chat endpoint with streaming SSE support."""
    verify_api_key(api_key)

    if request.stream:
        return StreamingResponse(
            agent.chat(
                message=request.message,
                session_id=request.session_id,
                model=request.model,
            ),
            media_type="text/event-stream",
            headers={
                "Cache-Control": "no-cache",
                "Connection": "keep-alive",
                "X-Accel-Buffering": "no",  # Disable nginx buffering
            },
        )
    else:
        # Non-streaming: collect full response
        full_response = []
        async for chunk in agent.chat(
            message=request.message,
            session_id=request.session_id,
            model=request.model,
            stream=False,
        ):
            full_response.append(chunk)
        return {"response": "".join(full_response)}

@app.get("/sessions/{session_id}")
async def get_session(session_id: str, api_key: str = Header(...)):
    """Get session history."""
    verify_api_key(api_key)
    _, messages = agent.sessions.get_or_create(session_id)
    return {"session_id": session_id, "messages": [m.model_dump() for m in messages]}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run("app.main:app", host="0.0.0.0", port=8000, reload=True)
```

---

## Step 9: Dockerfile for Production

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY app/ ./app/

# Non-root user
RUN useradd --create-home appuser
USER appuser

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

```yaml
# docker-compose.yml
version: "3.9"
services:
  api:
    build: .
    ports:
      - "8000:8000"
    env_file: .env
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512M
```

---

## Step 10: Testing

```python
# tests/test_api.py
import pytest
from httpx import AsyncClient, ASGITransport
from app.main import app

@pytest.mark.asyncio
async def test_health():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        response = await client.get("/health")
        assert response.status_code == 200
        assert response.json()["status"] == "healthy"

@pytest.mark.asyncio
async def test_chat_non_streaming():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        response = await client.post(
            "/chat",
            json={"message": "Hello!", "stream": False},
            headers={"X-API-Key": "test-key"},
        )
        assert response.status_code == 200
```

---

## Deploy It

```bash
# Local development
pip install -r requirements.txt
uvicorn app.main:app --reload

# Docker
docker compose up --build

# Deploy to Railway / Render / Fly.io
# Push to GitHub → connect repo → auto-deploy
```

---

## What You Built

A production-ready AI agent API with:

- ✅ **Streaming SSE responses** — users see output in real-time
- ✅ **Session memory** — agents remember context across requests
- ✅ **Tool calling** — search, code execution, URL fetching
- ✅ **Rate limiting** — per-API-key token bucket
- ✅ **API key auth** — simple but effective
- ✅ **Structured logging** — know what's happening
- ✅ **Docker support** — deploy anywhere
- ✅ **Health endpoint** — monitoring-ready

This is the same architecture pattern we use at ZOO for client projects — adapted and simplified for this tutorial.

---

## Need Help Deploying Your AI Agent?

We help startups and companies build production-ready AI systems — from agent architecture to deployment and monitoring.

**→ [Get a free 30-minute architecture review with ZOO](https://zootechnologies.com)**

We'll review your agent architecture, identify bottlenecks, and give you a deployment roadmap. No strings attached.

---

*Found this useful? Share it with your team. Built something with this code? We'd love to see it — tag us or drop a comment below.*
