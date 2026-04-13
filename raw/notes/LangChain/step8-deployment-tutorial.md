# Step 8: Deployment
## Deploy Agent ขึ้น Cloud ให้คนอื่นใช้ได้

> Step สุดท้าย! จากที่รันบนเครื่องตัวเอง → ทำเป็น API ให้ทุกคนเรียกใช้ได้

---

## 8.1 ตัวเลือกการ Deploy

```
ง่ายมาก (5 นาที):
  LangServe → เขียน 10 บรรทัด ได้ REST API
  → เหมาะ: prototype, demo

ง่าย (30 นาที):
  FastAPI + Railway/Render → deploy ฟรี
  → เหมาะ: project ส่วนตัว, MVP

ปานกลาง (2-3 ชั่วโมง):
  FastAPI + Docker + Cloud Run → production
  → เหมาะ: ทีมเล็ก, startup

Pro:
  LangGraph Platform → managed cloud
  → เหมาะ: production จริงจัง
```

---

## 8.2 วิธีที่ 1: LangServe (ง่ายที่สุด)

```bash
pip install "langserve[all]" uvicorn
```

```python
# file: 41_langserve.py

from fastapi import FastAPI
from langserve import add_routes
# add_routes คืออะไร?
# = function ที่แปลง LangChain chain เป็น REST API อัตโนมัติ
# ได้ /invoke, /batch, /stream, /playground ฟรี!

from langchain_google_genai import ChatGoogleGenerativeAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from dotenv import load_dotenv

load_dotenv()

model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")

# ==== สร้าง Chains ====
# Chain 1: แปลภาษา
translate = (
    ChatPromptTemplate.from_messages([
        ("system", "คุณคือนักแปล"),
        ("human", "แปลเป็น {language}: {text}"),
    ])
    | model | StrOutputParser()
)

# Chain 2: สรุป
summarize = (
    ChatPromptTemplate.from_messages([
        ("system", "คุณคือผู้เชี่ยวชาญสรุป"),
        ("human", "สรุปใน {sentences} ประโยค:\n{text}"),
    ])
    | model | StrOutputParser()
)

# Chain 3: Q&A
qa = (
    ChatPromptTemplate.from_messages([
        ("system", "คุณคือผู้ช่วย AI ตอบภาษาไทย สั้นกระชับ"),
        ("human", "{question}"),
    ])
    | model | StrOutputParser()
)

# ==== สร้าง FastAPI App ====
app = FastAPI(
    title="🤖 AI API Server",
    version="1.0",
    description="LangChain API พร้อมใช้งาน",
)

# ==== เพิ่ม Routes ====
add_routes(app, translate, path="/translate")
add_routes(app, summarize, path="/summarize")
add_routes(app, qa, path="/qa")
# แค่ 3 บรรทัดนี้ ได้ 3 APIs + Swagger docs + Playground!

# ==== รัน ====
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
    #                 ▲                  ▲
    #           "0.0.0.0" = รับ           port 8000
    #           connection จากทุก IP
```

### รัน

```bash
python 41_langserve.py

# Output:
# INFO:     Uvicorn running on http://0.0.0.0:8000
```

### สิ่งที่ได้อัตโนมัติ

```
เปิด browser:

http://localhost:8000/docs              ← Swagger UI (API docs)
http://localhost:8000/translate/playground ← ลองแปลภาษา
http://localhost:8000/summarize/playground ← ลองสรุป
http://localhost:8000/qa/playground        ← ลองถาม-ตอบ
```

### เรียก API

```python
# file: 42_call_api.py

import requests

# ==== invoke ====
response = requests.post(
    "http://localhost:8000/translate/invoke",
    json={
        "input": {
            "language": "English",
            "text": "สวัสดีครับ"
        }
    }
)
print(response.json())
# {"output": "Hello"}

# ==== batch ====
response = requests.post(
    "http://localhost:8000/qa/batch",
    json={
        "inputs": [
            {"question": "1+1=?"},
            {"question": "Python คืออะไร"},
        ]
    }
)
print(response.json())
# {"output": ["2", "Python เป็นภาษาโปรแกรม..."]}
```

---

## 8.3 วิธีที่ 2: FastAPI เอง (ควบคุมได้มากกว่า)

```python
# file: 43_fastapi_server.py

import os
import uuid
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
from dotenv import load_dotenv
from langchain_google_genai import ChatGoogleGenerativeAI

load_dotenv()

model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

# ==== เก็บ session (ในงานจริงใช้ Redis/Database) ====
sessions: dict = {}

# ==== Request/Response Models ====
class ChatRequest(BaseModel):
    message: str
    session_id: str | None = None
    # str | None = ใส่หรือไม่ใส่ก็ได้
    # ถ้าไม่ใส่ → สร้าง session ใหม่

class ChatResponse(BaseModel):
    session_id: str
    message: str

# ==== App ====
app = FastAPI(title="Chat API")

# CORS — ให้ frontend เรียกได้
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],      # production: ระบุ domain จริง
    allow_methods=["*"],
    allow_headers=["*"],
)
# CORSMiddleware คืออะไร?
# = ตั้งค่าให้ browser จาก domain อื่น (เช่น frontend)
#   สามารถเรียก API นี้ได้
# ถ้าไม่มี → frontend จะโดน CORS error

# ==== Health Check ====
@app.get("/health")
async def health():
    return {"status": "ok"}
# Health check คืออะไร?
# = endpoint ง่ายๆ ที่บอกว่า server ยังทำงานอยู่ไหม
# Cloud providers ใช้ตรวจสอบว่า server ยังอยู่

# ==== Chat Endpoint ====
@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    # จัดการ session
    sid = request.session_id or str(uuid.uuid4())
    #                           ▲
    #                   สร้าง ID ใหม่ถ้าไม่มี

    if sid not in sessions:
        sessions[sid] = []

    # เพิ่ม message
    sessions[sid].append({"role": "user", "content": request.message})

    # สร้าง prompt
    messages = [
        {"role": "system", "content": "คุณคือผู้ช่วย AI ตอบภาษาไทย สั้นกระชับ"},
        *sessions[sid][-20:]  # เก็บแค่ 20 messages ล่าสุด (ป้องกัน token เกิน)
    ]

    try:
        response = model.invoke(messages)
        sessions[sid].append({"role": "assistant", "content": response.content})
        return ChatResponse(session_id=sid, message=response.content)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# ==== Stream Endpoint ====
@app.post("/chat/stream")
async def chat_stream(request: ChatRequest):
    sid = request.session_id or str(uuid.uuid4())
    if sid not in sessions:
        sessions[sid] = []
    sessions[sid].append({"role": "user", "content": request.message})

    messages = [
        {"role": "system", "content": "คุณคือผู้ช่วย AI"},
        *sessions[sid][-20:]
    ]

    async def generate():
        full = ""
        for chunk in model.stream(messages):
            if chunk.content:
                full += chunk.content
                yield f"data: {chunk.content}\n\n"
                # ▲ Server-Sent Events (SSE) format
                # frontend ใช้ EventSource API รับ
        sessions[sid].append({"role": "assistant", "content": full})
        yield "data: [DONE]\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## 8.4 Docker — บรรจุเป็น Container

```dockerfile
# file: Dockerfile

FROM python:3.11-slim
# ▲ ใช้ Python 3.11 image ที่เล็ก

WORKDIR /app
# ▲ ตั้ง working directory

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
# ▲ ติดตั้ง dependencies

COPY . .
# ▲ copy code ทั้งหมดเข้า container

EXPOSE 8000
# ▲ บอกว่า container ใช้ port 8000

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1
# ▲ ตรวจสอบทุก 30 วินาทีว่า server ยังอยู่

CMD ["uvicorn", "43_fastapi_server:app", "--host", "0.0.0.0", "--port", "8000"]
# ▲ คำสั่งรัน server
```

```bash
# สร้าง requirements.txt
pip freeze > requirements.txt

# Build image
docker build -t my-ai-api .
# -t my-ai-api = ตั้งชื่อ image

# Run container
docker run -p 8000:8000 --env-file .env my-ai-api
# -p 8000:8000      = map port
# --env-file .env   = ส่ง environment variables เข้า container
```

---

## 8.5 Deploy ขึ้น Cloud (ฟรี)

### Railway (ง่ายที่สุด)

```bash
# 1. ติดตั้ง Railway CLI
npm install -g @railway/cli

# 2. Login
railway login

# 3. Deploy!
railway up
# Railway จะ:
# - detect Dockerfile อัตโนมัติ
# - build image
# - deploy ให้
# - ให้ URL มา เช่น https://my-ai-api.up.railway.app
```

### Render (ฟรี)

```yaml
# render.yaml
services:
  - type: web
    name: my-ai-api
    runtime: docker
    plan: free
    envVars:
      - key: GOOGLE_API_KEY
        sync: false  # ← ใส่ค่าบน Render dashboard
```

### Google Cloud Run

```bash
# 1. Build & Push image
gcloud builds submit --tag gcr.io/MY_PROJECT/my-ai-api

# 2. Deploy
gcloud run deploy my-ai-api \
  --image gcr.io/MY_PROJECT/my-ai-api \
  --port 8000 \
  --memory 512Mi \
  --set-env-vars "GOOGLE_API_KEY=AIza..." \
  --allow-unauthenticated
```

---

## 8.6 Frontend เชื่อม API (ตัวอย่าง HTML)

```html
<!-- file: index.html -->
<!DOCTYPE html>
<html>
<body>
  <h1>🤖 AI Chat</h1>
  <div id="chat"></div>
  <input id="input" placeholder="พิมพ์ข้อความ..." />
  <button onclick="send()">ส่ง</button>

  <script>
    let sessionId = null;

    async function send() {
      const input = document.getElementById('input');
      const chat = document.getElementById('chat');
      const message = input.value;
      input.value = '';

      chat.innerHTML += `<p><b>You:</b> ${message}</p>`;

      const response = await fetch('http://localhost:8000/chat', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({message, session_id: sessionId})
      });

      const data = await response.json();
      sessionId = data.session_id;
      chat.innerHTML += `<p><b>AI:</b> ${data.message}</p>`;
    }
  </script>
</body>
</html>
```

---

## 8.7 Production Checklist

```
📋 ก่อน Deploy:
  ☐ API keys อยู่ใน .env (ไม่ hardcode)
  ☐ .env อยู่ใน .gitignore
  ☐ Health check endpoint ทำงาน
  ☐ CORS ตั้งค่าเฉพาะ domain ที่ต้องการ (ไม่ใช่ *)
  ☐ Input validation (จำกัดความยาว message)
  ☐ Error handling ครบ (try/except)
  ☐ Rate limiting (ป้องกันถูก spam)
  ☐ temperature=0 สำหรับ agent/RAG

📋 หลัง Deploy:
  ☐ ทดสอบ API จาก URL จริง
  ☐ เปิด Observability (Phoenix / LangFuse)
  ☐ รัน Evaluation กับ test cases
  ☐ ตั้ง alert สำหรับ error rate สูง
```

---

## สรุป Step 8

```
✅ LangServe — สร้าง REST API อัตโนมัติ (ง่ายที่สุด)
✅ FastAPI — สร้าง API เอง (ควบคุมได้มากกว่า)
✅ Streaming endpoint (Server-Sent Events)
✅ Session management (จำบทสนทนา)
✅ Docker — บรรจุเป็น container
✅ Deploy ขึ้น Cloud (Railway / Render / Cloud Run)
✅ Frontend เชื่อม API
✅ Production Checklist
```

---

## 🎉 จบครบทุก Step!

```
Step 1: LangChain พื้นฐาน     ✅ Model, Messages, Prompts, Chains, Structured Output
Step 2: Tools & Agents         ✅ @tool, bind_tools, create_agent, ReAct Loop
Step 3: RAG                    ✅ Load → Split → Embed → Store → Retrieve → Generate
Step 4: LangGraph              ✅ State, Nodes, Edges, Checkpointing, Human-in-the-Loop
Step 5: Advanced RAG           ✅ Hybrid Search, Re-ranking, Multi-query, Conversational RAG
Step 6: Multi-Agent            ✅ Supervisor, Handoff, Pipeline patterns
Step 7: Evaluation             ✅ Phoenix, LangFuse, Rule-based, LLM-as-Judge
Step 8: Deployment             ✅ LangServe, FastAPI, Docker, Cloud Deploy

ไฟล์ทั้งหมด: 01_hello.py → 43_fastapi_server.py (43 ไฟล์)
```
