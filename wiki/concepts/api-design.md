---
title: "API Design — ออกแบบ REST API"
type: concept
tags: [api, rest, fastapi, design, http, endpoints]
sources: [wiki/sources/rag-integration-guide-complete, wiki/sources/deployment-deep-dive, wiki/sources/openrag-sdk-reference]
related: [wiki/concepts/rag-retrieval-augmented-generation, wiki/concepts/ai-agent, wiki/concepts/api-integration]
created: 2026-04-14
updated: 2026-04-14
---

## สรุปสั้น

API Design คือการออกแบบ interface สำหรับ REST API ที่ดี — มี endpoint ที่สื่อความหมาย, response ที่ consistent, error handling ที่ชัดเจน และ authentication ที่เหมาะสม สำคัญมากสำหรับ RAG Agent ที่ต้องการให้ channel ต่างๆ เชื่อมต่อได้

## อธิบาย

### REST API Principles

```
GET    /documents          — ดูรายการ
POST   /documents          — สร้างใหม่
GET    /documents/{id}     — ดูชิ้นเดียว
PUT    /documents/{id}     — แทนที่ทั้งหมด
PATCH  /documents/{id}     — อัปเดตบางส่วน
DELETE /documents/{id}     — ลบ
```

### FastAPI สำหรับ RAG Agent

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI(title="Sellsuki RAG Agent")

class ChatRequest(BaseModel):
    message: str
    session_id: str | None = None
    user_id: str | None = None

class ChatResponse(BaseModel):
    answer: str
    sources: list[str]
    confidence: float

@app.post("/chat", response_model=ChatResponse)
async def chat(req: ChatRequest):
    result = await agent.ainvoke({"input": req.message})
    return ChatResponse(
        answer=result["output"],
        sources=result.get("sources", []),
        confidence=result.get("confidence", 0.8)
    )
```

## ประเด็นสำคัญ

### Endpoints ที่ RAG Agent ต้องการ

| Endpoint | Method | ใช้สำหรับ |
|---------|--------|---------|
| `/chat` | POST | ส่งคำถาม รับคำตอบ |
| `/chat/stream` | POST | Streaming response (แสดงทีละคำ) |
| `/search` | GET | ค้นหาเอกสาร (debug) |
| `/ingest` | POST | เพิ่มเอกสารใหม่ |
| `/documents` | GET | ดูรายการเอกสาร |
| `/health` | GET | Health check |

### Authentication

```python
from fastapi.security import HTTPBearer

security = HTTPBearer()

@app.post("/chat")
async def chat(req: ChatRequest, token: str = Depends(security)):
    # ตรวจ JWT token
    payload = jwt.decode(token.credentials, SECRET_KEY)
    user_id = payload["sub"]
```

### Error Response Format (Consistent)

```json
{
  "error": {
    "code": "KNOWLEDGE_NOT_FOUND",
    "message": "ไม่พบข้อมูลที่ตรงกับคำถามนี้",
    "details": {"query": "..."}
  }
}
```

## ตัวอย่าง / กรณีศึกษา

**OpenRAG SDK:** TypeScript/Python client ที่ wrap REST API — `client.chat.query()`, `client.documents.upload()`, `client.search.query()` — ตัวอย่าง API design ที่ดีของ RAG platform

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/api-integration|API Integration]] — หลัง design API แล้ว channel ต่างๆ จะ integrate ผ่าน API นี้
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — REST API เป็น interface หลักที่ expose RAG Agent ออกมา
- [[wiki/concepts/ai-agent|AI Agent]] — Agent ทำงานอยู่ข้างหลัง API endpoint

## แหล่งที่มา

- [[wiki/sources/rag-integration-guide-complete|RAG Agent Integration — Deploy แล้วต่ออะไรได้บ้าง]]
- [[wiki/sources/deployment-deep-dive|Deployment Deep Dive]]
- [[wiki/sources/openrag-sdk-reference|OpenRAG SDK Reference]]
