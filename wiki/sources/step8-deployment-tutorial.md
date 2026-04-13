---
title: "Step 8: Deployment"
type: source
source_file: raw/notes/LangChain/step8-deployment-tutorial.md
tags: [langchain, deployment, langserve, fastapi, docker, cloud]
related: []
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/LangChain/step8-deployment-tutorial.md|Original file]]

## สรุป

บทช่วยสอน Deployment สุดท้ายของ series สอนวิธี Deploy Agent ขึ้น Cloud ให้คนอื่นใช้ได้ เนื้อหาครอบคลุม LangServe (ง่ายที่สุด), FastAPI (ควบคุมได้มากกว่า), Docker, Deploy ขึ้น Cloud (Railway / Render / Cloud Run), และ Production Checklist

## ประเด็นสำคัญ

### ตัวเลือกการ Deploy
- **ง่ายมาก (5 นาที)**: LangServe → เขียน 10 บรรทัด ได้ REST API → เหมาะ: prototype, demo
- **ง่าย (30 นาที)**: FastAPI + Railway/Render → deploy ฟรี → เหมาะ: project ส่วนตัว, MVP
- **ปานกลาง (2-3 ชั่วโมง)**: FastAPI + Docker + Cloud Run → production → เหมาะ: ทีมเล็ก, startup
- **Pro**: LangGraph Platform → managed cloud → เหมาะ: production จริงจัง

### LangServe (ง่ายที่สุด)
```bash
pip install "langserve[all]" uvicorn
```
```python
from fastapi import FastAPI
from langserve import add_routes

add_routes(app, translate, path="/translate")
add_routes(app, summarize, path="/summarize")
add_routes(app, qa, path="/qa")
# แค่ 3 บรรทัดนี้ ได้ 3 APIs + Swagger docs + Playground!
```
- ได้อัตโนมัติ:
  - `/docs` → Swagger UI (API docs)
  - `/translate/playground` → ลองแปลภาษา
  - `/translate/invoke` → REST API endpoint
  - `/translate/batch` → Batch endpoint
  - `/translate/stream` → Streaming endpoint

### FastAPI เอง (ควบคุมได้มากกว่า)
```python
@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    # จัดการ session
    sid = request.session_id or str(uuid.uuid4())
    if sid not in sessions:
        sessions[sid] = []

    # เพิ่ม message
    sessions[sid].append({"role": "user", "content": request.message})

    # สร้าง prompt
    messages = [
        {"role": "system", "content": "คุณคือผู้ช่วย AI ตอบภาษาไทย สั้นกระชับ"},
        *sessions[sid][-20:]  # เก็บแค่ 20 messages ล่าสุด
    ]

    response = model.invoke(messages)
    sessions[sid].append({"role": "assistant", "content": response.content})
    return ChatResponse(session_id=sid, message=response.content)
```

### CORSMiddleware
- ตั้งค่าให้ browser จาก domain อื่น (เช่น frontend) สามารถเรียก API นี้ได้
- ถ้าไม่มี → frontend จะโดน CORS error
```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # production: ระบุ domain จริง
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### Health Check
- Endpoint ง่ายๆ ที่บอกว่า server ยังทำงานอยู่ไหม
- Cloud providers ใช้ตรวจสอบว่า server ยังอยู่
```python
@app.get("/health")
async def health():
    return {"status": "ok"}
```

### Streaming Endpoint
```python
@app.post("/chat/stream")
async def chat_stream(request: ChatRequest):
    async def generate():
        for chunk in model.stream(messages):
            if chunk.content:
                yield f"data: {chunk.content}\n\n"  # SSE format
        yield "data: [DONE]\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")
```

### Docker
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1
CMD ["uvicorn", "43_fastapi_server:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Deploy ขึ้น Cloud (ฟรี)
**Railway** (ง่ายที่สุด):
```bash
npm install -g @railway/cli
railway login
railway up  # Deploy!
```

**Render** (ฟรี):
```yaml
services:
  - type: web
    name: my-ai-api
    runtime: docker
    plan: free
    envVars:
      - key: GOOGLE_API_KEY
        sync: false
```

**Google Cloud Run**:
```bash
gcloud builds submit --tag gcr.io/MY_PROJECT/my-ai-api
gcloud run deploy my-ai-api \
  --image gcr.io/MY_PROJECT/my-ai-api \
  --port 8000 \
  --memory 512Mi \
  --set-env-vars "GOOGLE_API_KEY=AIza..." \
  --allow-unauthenticated
```

### Production Checklist
**ก่อน Deploy**:
- ☐ API keys อยู่ใน .env (ไม่ hardcode)
- ☐ .env อยู่ใน .gitignore
- ☐ Health check endpoint ทำงาน
- ☐ CORS ตั้งค่าเฉพาะ domain ที่ต้องการ (ไม่ใช่ *)
- ☐ Input validation (จำกัดความยาว message)
- ☐ Error handling ครบ (try/except)
- ☐ Rate limiting (ป้องกันถูก spam)
- ☐ temperature=0 สำหรับ agent/RAG

**หลัง Deploy**:
- ☐ ทดสอบ API จาก URL จริง
- ☐ เปิด Observability (Phoenix / LangFuse)
- ☐ รัน Evaluation กับ test cases
- ☐ ตั้ง alert สำหรับ error rate สูง

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- `Session management`: เก็บประวัติใน RAM (ในงานจริงใช้ Redis/Database)
- `Server-Sent Events (SSE)`: format สำหรับ streaming
- `0.0.0.0`: รับ connection จากทุก IP
- `HEALTHCHECK`: ตรวจสอบทุก 30 วินาทีว่า server ยังอยู่

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อมูลที่ขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/ai-agent.md|AI Agent]]
- [[wiki/concepts/agentic-rag.md|Agentic RAG]]
