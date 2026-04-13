# Deployment Deep Dive
## Deploy Agent ขึ้น Cloud ด้วย LangServe & LangGraph Cloud

> จากที่เราสร้าง Agent ใน local แล้ว ต่อไปคือการ deploy ขึ้น production ให้คนอื่นใช้ได้จริง

---

## สารบัญ

1. [Overview — ตัวเลือกในการ Deploy](#1-overview)
2. [LangServe — สร้าง REST API จาก Chain/Agent](#2-langserve)
3. [LangGraph Platform — Deploy Graph บน Cloud](#3-langgraph-platform)
4. [Deploy ด้วย FastAPI เอง](#4-fastapi)
5. [Deploy ด้วย Docker](#5-docker)
6. [Deploy บน Cloud Providers](#6-cloud-providers)
7. [Authentication & Security](#7-auth)
8. [Scaling & Performance](#8-scaling)
9. [Monitoring ใน Production](#9-monitoring)
10. [CI/CD Pipeline](#10-cicd)
11. [Cost Optimization](#11-cost)
12. [Production Checklist](#12-checklist)

---

## 1. Overview

### ตัวเลือกในการ Deploy

```
┌────────────────────────────────────────────────────────────┐
│                    Deployment Options                       │
├────────────────────┬───────────────────┬───────────────────┤
│   LangServe        │  LangGraph Cloud  │  Self-hosted      │
│   (REST API)       │  (Managed)        │  (FastAPI/Docker) │
├────────────────────┼───────────────────┼───────────────────┤
│ ✅ ง่ายมาก         │ ✅ ง่าย           │ ⚠️ ปานกลาง       │
│ ✅ Auto docs       │ ✅ Managed infra  │ ✅ ควบคุม 100%    │
│ ✅ Playground      │ ✅ Built-in state │ ✅ ไม่มีค่า vendor│
│ ⚠️ Limited scale   │ ✅ Auto-scaling   │ ⚠️ ดูแลเอง       │
│ ❌ ไม่มี state mgmt│ ✅ Monitoring     │ ⚠️ Scaling manual │
│                    │ 💰 มีค่าบริการ    │                   │
├────────────────────┼───────────────────┼───────────────────┤
│ เหมาะกับ:          │ เหมาะกับ:         │ เหมาะกับ:         │
│ Prototype, MVP     │ Production       │ Enterprise,       │
│ Simple chains      │ Complex agents   │ Custom infra      │
└────────────────────┴───────────────────┴───────────────────┘
```

---

## 2. LangServe

### 2.1 LangServe คืออะไร

LangServe แปลง LangChain chain/agent เป็น **REST API** อัตโนมัติ พร้อม Swagger docs และ Playground:

```bash
pip install langserve[all] uvicorn
```

### 2.2 สร้าง API Server แบบง่ายที่สุด

```python
# server.py
from fastapi import FastAPI
from langserve import add_routes
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from dotenv import load_dotenv

load_dotenv()

# สร้าง Chain
model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

# Chain 1: แปลภาษา
translate_chain = (
    ChatPromptTemplate.from_messages([
        ("system", "คุณคือนักแปลมืออาชีพ"),
        ("human", "แปลเป็น {language}: {text}"),
    ])
    | model
    | StrOutputParser()
)

# Chain 2: สรุปข้อความ
summarize_chain = (
    ChatPromptTemplate.from_messages([
        ("system", "คุณคือผู้เชี่ยวชาญด้านการสรุปข้อความ"),
        ("human", "สรุปข้อความนี้ใน {num_sentences} ประโยค:\n{text}"),
    ])
    | model
    | StrOutputParser()
)

# Chain 3: Q&A Bot
qa_chain = (
    ChatPromptTemplate.from_messages([
        ("system", "คุณคือผู้ช่วย AI ของบริษัท ABC ตอบสั้นกระชับเป็นภาษาไทย"),
        ("human", "{question}"),
    ])
    | model
    | StrOutputParser()
)

# สร้าง FastAPI app
app = FastAPI(
    title="AI API Server",
    version="1.0",
    description="LangChain API Server พร้อม Chains หลายตัว",
)

# เพิ่ม routes อัตโนมัติ
add_routes(app, translate_chain, path="/translate")
add_routes(app, summarize_chain, path="/summarize")
add_routes(app, qa_chain, path="/qa")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### 2.3 รัน Server

```bash
python server.py
# หรือ
uvicorn server:app --host 0.0.0.0 --port 8000 --reload
```

### 2.4 Endpoints ที่ได้อัตโนมัติ

```
เปิด browser ไปที่:
  http://localhost:8000/docs          ← Swagger UI (API docs)
  http://localhost:8000/translate/playground  ← Interactive Playground

แต่ละ chain จะได้ endpoints:
  POST /translate/invoke      ← เรียกใช้ chain
  POST /translate/batch       ← เรียกหลาย input พร้อมกัน
  POST /translate/stream      ← Stream ผลลัพธ์
  GET  /translate/input_schema   ← ดู input schema
  GET  /translate/output_schema  ← ดู output schema
```

### 2.5 เรียกใช้ API

```python
# client.py
import requests

# === invoke ===
response = requests.post(
    "http://localhost:8000/translate/invoke",
    json={"input": {"language": "English", "text": "สวัสดีครับ วันนี้อากาศดีมาก"}}
)
print(response.json())
# {"output": "Hello, the weather is very nice today."}

# === batch ===
response = requests.post(
    "http://localhost:8000/translate/batch",
    json={"inputs": [
        {"language": "English", "text": "สวัสดี"},
        {"language": "Japanese", "text": "สวัสดี"},
    ]}
)
print(response.json())
# {"output": ["Hello", "こんにちは"]}

# === stream ===
import httpx

with httpx.stream("POST", "http://localhost:8000/qa/stream",
    json={"input": {"question": "LangChain คืออะไร"}}) as r:
    for chunk in r.iter_text():
        print(chunk, end="")
```

### 2.6 ใช้ LangServe RemoteRunnable (Python Client)

```python
from langserve import RemoteRunnable

# เชื่อมต่อกับ remote chain เหมือนใช้ chain ปกติ
translate = RemoteRunnable("http://localhost:8000/translate")

# ใช้เหมือน chain ปกติเลย
result = translate.invoke({"language": "English", "text": "สวัสดี"})
print(result)  # "Hello"

# stream
for chunk in translate.stream({"language": "English", "text": "สวัสดี"}):
    print(chunk, end="")

# batch
results = translate.batch([
    {"language": "EN", "text": "สวัสดี"},
    {"language": "JP", "text": "สวัสดี"},
])
```

---

## 3. LangGraph Platform

### 3.1 LangGraph Platform คืออะไร

Managed infrastructure สำหรับ deploy LangGraph agents พร้อม built-in:
- State persistence (checkpointing)
- Async task queue
- Streaming support
- Cron jobs
- Double-texting handling (user ส่งหลายข้อความพร้อมกัน)

### 3.2 Setup

```bash
pip install -U langgraph-cli langgraph-sdk
```

### 3.3 โครงสร้าง Project

```
my-agent/
├── src/
│   └── agent.py           ← Agent code
├── langgraph.json          ← Configuration
├── .env                    ← API keys
└── requirements.txt
```

### 3.4 langgraph.json

```json
{
  "dependencies": ["."],
  "graphs": {
    "support_bot": "./src/agent.py:graph"
  },
  "env": ".env"
}
```

### 3.5 Agent Code

```python
# src/agent.py
import os
from typing import Annotated
from typing_extensions import TypedDict
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain.tools import tool
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.checkpoint.memory import MemorySaver

class State(TypedDict):
    messages: Annotated[list, add_messages]

@tool
def search_knowledge(query: str) -> str:
    """ค้นหาข้อมูลจาก knowledge base"""
    # ในงานจริงจะเชื่อมกับ vector store
    return f"ข้อมูลเกี่ยวกับ: {query}"

model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)
model_with_tools = model.bind_tools([search_knowledge])

def agent_node(state: State):
    response = model_with_tools.invoke(state["messages"])
    return {"messages": [response]}

builder = StateGraph(State)
builder.add_node("agent", agent_node)
builder.add_node("tools", ToolNode(tools=[search_knowledge]))
builder.add_edge(START, "agent")
builder.add_conditional_edges("agent", tools_condition)
builder.add_edge("tools", "agent")

# Export graph (ตัวนี้ที่ langgraph.json reference ถึง)
memory = MemorySaver()
graph = builder.compile(checkpointer=memory)
```

### 3.6 รัน Local Server

```bash
# ทดสอบ local
langgraph dev

# Output:
# 🚀 API: http://127.0.0.1:2024
# 📖 Docs: http://127.0.0.1:2024/docs
# 🎮 Studio: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
```

### 3.7 ใช้งานผ่าน SDK

```python
from langgraph_sdk import get_client
import asyncio

async def main():
    client = get_client(url="http://localhost:2024")

    # สร้าง thread (เหมือน session)
    thread = await client.threads.create()
    print(f"Thread ID: {thread['thread_id']}")

    # ส่งข้อความ (สร้าง run)
    run = await client.runs.create(
        thread_id=thread["thread_id"],
        assistant_id="support_bot",
        input={
            "messages": [{"role": "user", "content": "สวัสดีครับ ลาพักร้อนได้กี่วัน?"}]
        },
    )

    # รอผลลัพธ์
    result = await client.runs.join(thread["thread_id"], run["run_id"])
    print(f"Answer: {result['messages'][-1]['content']}")

    # ส่งข้อความต่อ (ต่อเนื่อง session เดิม)
    run2 = await client.runs.create(
        thread_id=thread["thread_id"],
        assistant_id="support_bot",
        input={
            "messages": [{"role": "user", "content": "แล้วลาป่วยล่ะ?"}]
        },
    )
    result2 = await client.runs.join(thread["thread_id"], run2["run_id"])
    print(f"Answer: {result2['messages'][-1]['content']}")

asyncio.run(main())
```

### 3.8 Stream Events ผ่าน SDK

```python
async def stream_example():
    client = get_client(url="http://localhost:2024")
    thread = await client.threads.create()

    async for event in client.runs.stream(
        thread_id=thread["thread_id"],
        assistant_id="support_bot",
        input={"messages": [{"role": "user", "content": "เล่าเรื่อง LangGraph"}]},
        stream_mode="events",
    ):
        if event.event == "on_chat_model_stream":
            print(event.data.get("chunk", {}).get("content", ""), end="")

asyncio.run(stream_example())
```

### 3.9 Deploy บน LangGraph Cloud

```bash
# Login
langgraph auth login

# Deploy
langgraph deploy --config langgraph.json

# Output:
# ✅ Deployed to: https://my-agent-abc123.langgraph.app
# 📖 API docs: https://my-agent-abc123.langgraph.app/docs
```

---

## 4. Deploy ด้วย FastAPI เอง

### 4.1 Server ที่ครบสมบูรณ์

```python
# server.py
import os
import uuid
from contextlib import asynccontextmanager
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
from dotenv import load_dotenv
from langchain_google_genai import ChatGoogleGenerativeAI

load_dotenv()

# ======== Models & Chains ========
model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

# ======== In-memory session storage ========
sessions: dict = {}

# ======== Request/Response Models ========
class ChatRequest(BaseModel):
    message: str
    session_id: str | None = None

class ChatResponse(BaseModel):
    session_id: str
    message: str
    tokens_used: int | None = None

# ======== App ========
app = FastAPI(title="AI Chat API", version="1.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # production: ระบุ domain จริง
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/health")
async def health():
    return {"status": "ok"}

@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    # จัดการ session
    session_id = request.session_id or str(uuid.uuid4())
    if session_id not in sessions:
        sessions[session_id] = []

    # เพิ่ม message
    sessions[session_id].append({"role": "user", "content": request.message})

    # สร้าง messages พร้อม system prompt
    messages = [
        {"role": "system", "content": "คุณคือผู้ช่วย AI ตอบเป็นภาษาไทย สั้นกระชับ"},
        *sessions[session_id][-20:]  # เก็บแค่ 20 messages ล่าสุด
    ]

    try:
        response = model.invoke(messages)
        sessions[session_id].append({"role": "assistant", "content": response.content})

        return ChatResponse(
            session_id=session_id,
            message=response.content,
            tokens_used=response.response_metadata.get("usage_metadata", {}).get("total_tokens"),
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/chat/stream")
async def chat_stream(request: ChatRequest):
    session_id = request.session_id or str(uuid.uuid4())
    if session_id not in sessions:
        sessions[session_id] = []

    sessions[session_id].append({"role": "user", "content": request.message})

    messages = [
        {"role": "system", "content": "คุณคือผู้ช่วย AI ตอบเป็นภาษาไทย"},
        *sessions[session_id][-20:]
    ]

    async def generate():
        full_response = ""
        for chunk in model.stream(messages):
            if chunk.content:
                full_response += chunk.content
                yield f"data: {chunk.content}\n\n"
        sessions[session_id].append({"role": "assistant", "content": full_response})
        yield f"data: [DONE]\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")

@app.delete("/session/{session_id}")
async def clear_session(session_id: str):
    if session_id in sessions:
        del sessions[session_id]
        return {"status": "cleared"}
    raise HTTPException(status_code=404, detail="Session not found")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## 5. Deploy ด้วย Docker

### 5.1 Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# ติดตั้ง dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy code
COPY . .

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

# Run
CMD ["uvicorn", "server:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 5.2 docker-compose.yml

```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8000:8000"
    env_file:
      - .env
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 1G

  # Vector Store (ถ้าใช้ RAG)
  chroma:
    image: chromadb/chroma:latest
    ports:
      - "8001:8000"
    volumes:
      - chroma_data:/chroma/chroma

volumes:
  chroma_data:
```

### 5.3 Build & Run

```bash
# Build
docker build -t ai-agent .

# Run
docker run -p 8000:8000 --env-file .env ai-agent

# หรือใช้ docker-compose
docker-compose up -d
```

---

## 6. Deploy บน Cloud Providers

### 6.1 Google Cloud Run

```bash
# Build & Push
gcloud builds submit --tag gcr.io/MY_PROJECT/ai-agent

# Deploy
gcloud run deploy ai-agent \
  --image gcr.io/MY_PROJECT/ai-agent \
  --port 8000 \
  --memory 1Gi \
  --cpu 2 \
  --min-instances 0 \
  --max-instances 10 \
  --set-env-vars "GOOGLE_API_KEY=xxx" \
  --allow-unauthenticated  # หรือ --no-allow-unauthenticated
```

### 6.2 AWS (ECS / Lambda)

```bash
# ECR + ECS
aws ecr create-repository --repository-name ai-agent
docker tag ai-agent:latest 123456789.dkr.ecr.ap-southeast-1.amazonaws.com/ai-agent
docker push 123456789.dkr.ecr.ap-southeast-1.amazonaws.com/ai-agent
# จากนั้นสร้าง ECS Task Definition + Service
```

### 6.3 Railway / Render / Fly.io (ง่ายที่สุด)

```bash
# Railway
railway up

# Render
# สร้าง render.yaml:
# services:
#   - type: web
#     name: ai-agent
#     runtime: docker
#     envVars:
#       - key: GOOGLE_API_KEY
#         sync: false

# Fly.io
fly launch
fly deploy
```

---

## 7. Authentication & Security

### 7.1 API Key Authentication

```python
from fastapi import FastAPI, Depends, HTTPException
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

VALID_API_KEYS = {"sk-abc123", "sk-xyz789"}  # production: ใช้ database

async def verify_api_key(credentials: HTTPAuthorizationCredentials = Depends(security)):
    if credentials.credentials not in VALID_API_KEYS:
        raise HTTPException(status_code=403, detail="Invalid API key")
    return credentials.credentials

@app.post("/chat")
async def chat(request: ChatRequest, api_key: str = Depends(verify_api_key)):
    # ... chat logic ...
    pass
```

### 7.2 Rate Limiting

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter

@app.post("/chat")
@limiter.limit("30/minute")  # 30 requests per minute per IP
async def chat(request: ChatRequest):
    pass
```

### 7.3 Input Validation & Sanitization

```python
class ChatRequest(BaseModel):
    message: str
    session_id: str | None = None

    @field_validator("message")
    def validate_message(cls, v):
        if len(v) > 10000:
            raise ValueError("Message too long (max 10000 chars)")
        if len(v.strip()) == 0:
            raise ValueError("Message cannot be empty")
        return v.strip()
```

---

## 8. Scaling & Performance

### 8.1 Caching

```python
from functools import lru_cache
import hashlib

# Cache embedding results
@lru_cache(maxsize=1000)
def cached_embed(text: str):
    return embeddings.embed_query(text)

# Cache LLM responses (สำหรับคำถามที่ถามบ่อย)
response_cache = {}

async def chat_with_cache(message: str):
    cache_key = hashlib.md5(message.encode()).hexdigest()
    if cache_key in response_cache:
        return response_cache[cache_key]

    response = model.invoke(message)
    response_cache[cache_key] = response.content
    return response.content
```

### 8.2 Async Workers

```python
# ใช้ async สำหรับ I/O-bound tasks
@app.post("/chat")
async def chat(request: ChatRequest):
    response = await model.ainvoke(messages)  # ← async invoke
    return ChatResponse(message=response.content)
```

---

## 9. Monitoring

### 9.1 Structured Logging

```python
import logging
import json
from datetime import datetime

logger = logging.getLogger("ai-api")

@app.post("/chat")
async def chat(request: ChatRequest):
    start_time = datetime.now()

    response = model.invoke(messages)

    duration = (datetime.now() - start_time).total_seconds()

    logger.info(json.dumps({
        "event": "chat_response",
        "session_id": session_id,
        "question_length": len(request.message),
        "answer_length": len(response.content),
        "duration_seconds": duration,
        "tokens": response.response_metadata.get("usage_metadata", {}),
    }))

    return ChatResponse(message=response.content)
```

### 9.2 LangSmith ใน Production

```env
LANGSMITH_TRACING=true
LANGSMITH_PROJECT=ai-agent-production
LANGSMITH_API_KEY=lsv2_pt_xxx
```

---

## 12. Production Checklist

```
📋 Pre-deployment:
  □ API keys อยู่ใน environment variables (ไม่ hardcode)
  □ .env อยู่ใน .gitignore
  □ Health check endpoint ทำงาน
  □ Input validation ครบถ้วน
  □ Rate limiting ตั้งค่าแล้ว
  □ Error handling ครอบคลุม
  □ Logging ตั้งค่าแล้ว
  □ CORS ตั้งค่าเฉพาะ domain ที่ต้องการ

📋 Deployment:
  □ Docker image build สำเร็จ
  □ Environment variables ตั้งค่าบน cloud
  □ Health check ทำงานหลัง deploy
  □ SSL/HTTPS เปิดใช้งาน
  □ Auto-scaling ตั้งค่าแล้ว (ถ้าจำเป็น)

📋 Post-deployment:
  □ LangSmith tracing เปิดใช้งาน
  □ Monitoring dashboard ตั้งค่า
  □ Alert ตั้งค่า (error rate, latency)
  □ ทดสอบ API จาก production URL
  □ Load test (ถ้าจำเป็น)

📋 Ongoing:
  □ รัน evaluation สม่ำเสมอ
  □ Review traces ใน LangSmith
  □ อัปเดต dependencies
  □ Monitor cost
  □ Review & rotate API keys
```
