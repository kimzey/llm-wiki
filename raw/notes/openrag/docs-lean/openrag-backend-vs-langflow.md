# Backend ทำอะไร vs Langflow ทำอะไร — แบ่งหน้าที่ชัดเจน

> **คำตอบสั้น:** Logic หลักแบ่งครึ่ง — Backend เป็น "โครงสร้างพื้นฐาน + ความปลอดภัย"
> Langflow เป็น "AI pipeline ที่ปรับแต่งได้" ทั้งสองทำงานร่วมกัน ขาดกันไม่ได้

---

## ภาพรวม: แต่ละชั้นทำอะไร

```
┌─────────────────────────────────────────────────────────────────────┐
│  Frontend (Next.js :3000)                                           │
│  UI, routing, state management                                      │
└────────────────────┬────────────────────────────────────────────────┘
                     │ HTTP / SSE
┌────────────────────▼────────────────────────────────────────────────┐
│  BACKEND (FastAPI :8000)  ← "กำแพงด้านนอก + ผู้ควบคุม"            │
│                                                                     │
│  ✅ Authentication & Authorization (JWT, OAuth, API Key)            │
│  ✅ Session & User Management                                       │
│  ✅ Task Queue & Async Processing                                   │
│  ✅ Cloud Connector (Google Drive, OneDrive, SharePoint)            │
│  ✅ Document ACL Management                                         │
│  ✅ OpenSearch Index Setup & OIDC config                           │
│  ✅ Knowledge Filter CRUD                                           │
│  ✅ Settings & Configuration                                        │
│  ✅ Telemetry & Monitoring                                          │
│  ✅ Flow Backup & Recovery                                          │
│  ✅ Direct Agent mode (ไม่ผ่าน Langflow)                           │
│                                                                     │
│  → เรียกใช้ Langflow ผ่าน HTTP สำหรับ AI pipeline                  │
└────────────────────┬────────────────────────────────────────────────┘
                     │ HTTP (internal)
┌────────────────────▼────────────────────────────────────────────────┐
│  LANGFLOW (:7860)  ← "AI Pipeline Engine ที่ปรับแต่งได้"           │
│                                                                     │
│  ✅ Document Parsing (via Docling)                                  │
│  ✅ Text Splitting / Chunking                                       │
│  ✅ Embedding (OpenAI, Ollama, WatsonX)                             │
│  ✅ Vector Store Write (OpenSearch)                                 │
│  ✅ RAG Query + LLM Response                                        │
│  ✅ Agent Orchestration (Tool use, reasoning)                       │
│  ✅ MCP Server integration                                          │
│  ✅ Prompt Template management                                      │
│                                                                     │
└────────────────────┬────────────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────────────┐
│  OPENSEARCH (:9200)  ← "Vector Database + ACL Enforcement"          │
│  Store, Index, KNN Search, DLS Filter                               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Backend ทำอะไรบ้าง (ทั้งหมด)

### 1. Authentication & Authorization

Backend เป็น **identity provider** ของระบบทั้งหมด

```
OAuth Login (Google / Microsoft)
  → auth_service.py → session_manager.py
  → สร้าง JWT (RS256, 7 วัน)
  → set httponly cookie "auth_token"

API Key Authentication
  → api_key_service.py
  → เก็บ SHA-256 hash ใน OpenSearch index "api_keys"
  → prefix: orag_xxxxx

JWT ทำหน้าที่ 2 อย่างพร้อมกัน:
  1. authen user ใน Backend
  2. ส่งต่อให้ OpenSearch เป็น Bearer token → OIDC DLS filter
```

**ไฟล์หลัก:** `src/api/auth.py`, `src/session_manager.py`, `src/dependencies.py`, `src/services/api_key_service.py`

---

### 2. Task Queue & Async Processing

Backend จัดการ **งานหนักทั้งหมดแบบ async** — user ไม่ต้อง wait

```
POST /router/upload_ingest
  ↓
TaskService.create_langflow_upload_task()
  ↓
Return { task_id: "abc123" } ← ทันที (202 Accepted)
  ↓
Background: process each file → Langflow → index
  ↓
GET /tasks/{task_id} ← ตรวจสอบสถานะ
```

**TaskService ควบคุม:**
- Global semaphore จำกัด concurrent worker (ตาม GPU/CPU)
- Timeout (default 3600 วินาที/ไฟล์)
- Periodic cleanup (ทุก 2 ชั่วโมง)
- Cancel task

**ไฟล์หลัก:** `src/services/task_service.py`, `src/models/tasks.py`

---

### 3. Cloud Connectors (Google Drive / OneDrive / SharePoint)

Backend จัดการ OAuth flow และ Webhook ทั้งหมด — Langflow ไม่รู้จัก connector เหล่านี้เลย

```
Google Drive:
  1. User กด Connect → Backend redirect ไป Google OAuth
  2. Google callback → Backend เก็บ tokens
  3. User sync → Backend list files → download → ส่งให้ Langflow ingest
  4. Google webhook → Backend รับ → trigger re-index เฉพาะไฟล์นั้น

ACL extraction:
  → _extract_acl() → permissions().list() API
  → type=="user" → allowed_users[]
  → type=="group" → allowed_groups[]
  → ส่งผ่าน HTTP headers ไปให้ Langflow
```

**ไฟล์หลัก:** `src/connectors/google_drive/`, `src/connectors/onedrive/`, `src/connectors/sharepoint/`, `src/connectors/service.py`

---

### 4. OpenSearch Management

Backend เป็นผู้สร้างและดูแล OpenSearch index ทั้งหมด

```
Startup:
  → สร้าง index "documents" ด้วย dynamic vector dimensions
  → สร้าง index "knowledge_filters"
  → สร้าง index "api_keys"
  → configure DLS security roles (securityconfig/)
  → configure alerting settings

OIDC Setup:
  → Backend เปิด endpoint /.well-known/openid-configuration
  → OpenSearch ใช้ endpoint นี้ validate JWT ของ user
  → ทุกครั้ง user query → OpenSearch validate JWT → apply DLS filter
```

**ไฟล์หลัก:** `src/main.py (init_index)`, `securityconfig/roles.yml`, `src/api/oidc.py`

---

### 5. Settings & Configuration Management

Backend เก็บ config ทุกอย่างและ **push settings เข้า Langflow** โดยอัตโนมัติ

```
User เปลี่ยน LLM model ใน UI
  ↓
POST /settings → Backend บันทึก config
  ↓
Backend inject settings ไปที่ Langflow:
  - update global variables ใน Langflow flows
  - X-LANGFLOW-GLOBAL-VAR-SELECTED_EMBEDDING_MODEL
  - X-LANGFLOW-GLOBAL-VAR-LLM_MODEL
  - ... (provider API keys, etc.)

เมื่อ Startup:
  → check_flows_reset() ← ถ้า Langflow reset → reapply all settings
  → periodic backup ทุก 5 นาที ← backup flow JSON ไว้กันสูญหาย
```

**ไฟล์หลัก:** `src/api/settings.py`, `src/services/flows_service.py`

---

### 6. Knowledge Filter (การกรองความรู้ตาม context)

ระบบ Knowledge Filter ทั้งหมดอยู่ใน Backend — ไม่ผ่าน Langflow

```
Knowledge Filter = saved search query ที่ใช้กรอง documents ก่อน RAG
ตัวอย่าง: filter "hr-only" → เฉพาะเอกสาร HR

CRUD: POST/GET/PUT/DELETE /knowledge-filter
Subscribe: user subscribe ต่อ filter
Webhook: OpenSearch notify เมื่อ documents ใน filter เปลี่ยน
```

**ไฟล์หลัก:** `src/api/knowledge_filter.py`, `src/services/knowledge_filter_service.py`

---

### 7. Direct Agent Mode (ไม่ผ่าน Langflow)

Backend มี AI agent ของตัวเอง — ใช้เมื่อ `DISABLE_INGEST_WITH_LANGFLOW=true` หรือ `/chat` endpoint

```python
# agent.py — system prompt ฝังอยู่ใน Backend โดยตรง
system_prompt = """You are the OpenRAG Agent...
Available Tools:
- OpenSearch Retrieval Tool
- URL Ingestion Tool
- Calculator Tool
"""

# tools ใน SearchService
@tool
async def search_tool(self, query: str) → dict:
    # hybrid KNN + BM25 search
    # ใช้ JWT ของ user → OpenSearch DLS filter อัตโนมัติ
```

**ไฟล์หลัก:** `src/agent.py`, `src/services/search_service.py`, `src/services/chat_service.py`

---

### 8. Monitoring & Health

```
GET /health                 ← liveness probe
GET /search/health          ← readiness probe (ping OpenSearch)
GET /docling/health         ← check Docling service
GET /provider/health        ← check LLM/embedding provider
```

---

## Langflow ทำอะไรบ้าง

Langflow รับผิดชอบ **AI pipeline ล้วนๆ** — ไม่รู้จัก user, ไม่จัดการ auth

### Flow 1: ingestion_flow — Document Processing Pipeline

```
[TextInput nodes] ← รับ metadata จาก Backend headers
  OWNER, OWNER_EMAIL, ALLOWED_USERS, ALLOWED_GROUPS, ...
      │
      ▼
[DoclingRemote] ← ส่ง file ไปให้ Docling parse
  → PDF/DOCX/PPTX → structured text + tables + images
      │
      ▼
[SplitText] ← แบ่ง text เป็น chunks
  → chunk_size, chunk_overlap configurable
      │
      ▼
[EmbeddingModel] ← แปลง text → vector
  → OpenAI / Ollama / WatsonX (ตาม config)
      │
      ▼
[OpenSearchVectorStore] ← เขียน chunks + vectors + ACL metadata
  → index: "documents"
```

**สิ่งที่ Langflow ทำเอง:** parse, split, embed, write
**สิ่งที่ Langflow ไม่รู้:** user คือใคร, permission ถูกหรือเปล่า, files มาจากไหน

---

### Flow 2: openrag_agent — Chat RAG Pipeline

```
[ChatInput] ← รับ prompt จาก user
      │
      ▼
[Agent node] ← LLM orchestrator
  ├── [OpenSearchVectorStore] ← vector search
  │     ↳ JWT → OIDC → DLS filter อัตโนมัติ
  ├── [Calculator] ← math tool
  ├── [MCP servers] ← external tools
  └── [Prompt Template] ← system instructions
      │
      ▼
[ChatOutput] ← ส่ง response กลับ
```

**สิ่งที่ Langflow ทำเอง:** reasoning, tool selection, retrieval, LLM call
**สิ่งที่ Langflow รับจาก Backend:** JWT token (via headers) → ส่งต่อให้ OpenSearch

---

### Flow 3: openrag_nudges — Smart Suggestions

```
[TextInput] ← รับ context จาก Backend
      │
      ▼
[LLM] ← generate suggested questions
      │
      ▼
[TextOutput] ← ส่ง nudges กลับ
```

---

## เส้นทางข้อมูล: จาก User ถึง OpenSearch

```
User upload file
      │
      ▼
[Backend: POST /router/upload_ingest]
  → authenticate user (JWT/API Key)
  → extract user_id, email, name
  → save to temp file
  → create TaskService task
      │
      ▼
[Backend → Langflow: POST /api/v1/run/{flow_id}]
  Headers inject:
    X-LANGFLOW-GLOBAL-VAR-OWNER = user_id
    X-LANGFLOW-GLOBAL-VAR-OWNER_EMAIL = email
    X-LANGFLOW-GLOBAL-VAR-ALLOWED_USERS = []
    X-LANGFLOW-GLOBAL-VAR-ALLOWED_GROUPS = []
    X-LANGFLOW-GLOBAL-VAR-JWT = jwt_token
    ...
  Body: { file upload }
      │
      ▼
[Langflow: ingestion_flow runs]
  → Docling parse
  → SplitText
  → Embed
  → Write to OpenSearch (with ACL fields from headers)
      │
      ▼
[OpenSearch: document stored]
  {
    "text": "...",
    "owner": "user_id_from_header",
    "allowed_users": [],
    "allowed_groups": [],
    "chunk_embedding": [0.1, 0.2, ...]
  }
```

---

## เส้นทางข้อมูล: User query → RAG response

```
User chat: "What is Q3 budget?"
      │
      ▼
[Backend: POST /langflow]
  → authenticate user
  → get jwt_token
  → inject headers: JWT, EMBEDDING_MODEL, FILTER
      │
      ▼
[Langflow: openrag_agent flow]
  Agent node decides → call search tool
  → POST to OpenSearch KNN query
    Headers: Authorization: Bearer {jwt_token}
      │
      ▼
[OpenSearch]
  DLS filter runs (from securityconfig/roles.yml):
    match owner == user OR allowed_users has user
  → return only authorized chunks
      │
      ▼
[Langflow: Agent synthesizes answer]
      │
      ▼
[Backend: stream SSE back to Frontend]
      │
      ▼
User sees answer
```

---

## ตารางสรุป: Logic อยู่ที่ไหน

| ฟีเจอร์ | Backend | Langflow | หมายเหตุ |
|---------|---------|----------|----------|
| Login / OAuth | ✅ | ❌ | Backend เท่านั้น |
| JWT สร้าง/validate | ✅ | ❌ | |
| API Key management | ✅ | ❌ | |
| Document ACL (owner, allowed_users) | ✅ set | ✅ store | Backend set → Langflow เขียนลง OS |
| Document parse (PDF→text) | ❌ | ✅ | Langflow เรียก Docling |
| Text splitting | ❌ | ✅ | SplitText node |
| Embedding | ❌ | ✅ | EmbeddingModel node |
| Vector index write | ❌ | ✅ | OpenSearchVectorStore node |
| RAG retrieval | ❌ | ✅ | KNN search ใน flow |
| LLM call / Agent reasoning | ❌ | ✅ | Agent node |
| DLS security enforce | ❌ | ❌ | OpenSearch เอง (via roles.yml) |
| Task queue | ✅ | ❌ | TaskService |
| Google Drive sync + webhook | ✅ | ❌ | Connector service |
| OneDrive / SharePoint sync | ✅ | ❌ | Connector service |
| S3 upload | ✅ | ❌ | |
| Settings management | ✅ | ❌ | backend push ไปให้ Langflow |
| Flow backup/restore | ✅ | ❌ | FlowsService ทุก 5 นาที |
| Knowledge filters | ✅ | ❌ | |
| Chat history | ✅ | ✅ | Backend (direct mode) / Langflow |
| Conversation threading | ✅ | ❌ | agent.py |
| Health monitoring | ✅ | ❌ | |
| MCP servers | ❌ | ✅ | nodes ใน flows |

---

## คำตอบชัดๆ: Logic อยู่ไหน?

```
Backend = Infrastructure + Security + Orchestration

  ┌─ ทุกอย่างที่เกี่ยวกับ "ใคร" → Backend
  │   login, permission, user identity, ACL
  │
  ├─ ทุกอย่างที่เกี่ยวกับ "จัดการ" → Backend
  │   task queue, connector sync, settings, backup
  │
  └─ ทุกอย่างที่เกี่ยวกับ "ปลอดภัย" → Backend + OpenSearch
      JWT, DLS filter, API keys


Langflow = AI Processing Pipeline

  ┌─ ทุกอย่างที่เกี่ยวกับ "ประมวลผลเอกสาร" → Langflow
  │   parse, split, embed, index
  │
  ├─ ทุกอย่างที่เกี่ยวกับ "AI reasoning" → Langflow
  │   LLM call, agent, tool use, prompt
  │
  └─ ทุกอย่างที่ "ปรับแต่งได้ไม่ต้องแก้โค้ด" → Langflow
      เพิ่ม node, เปลี่ยน flow, เชื่อม MCP ใหม่


ข้อสรุปสำคัญ:
  → Backend ไม่รู้จัก AI — มันแค่เรียก Langflow
  → Langflow ไม่รู้จัก user — มันแค่รับ metadata จาก headers
  → OpenSearch บังคับ security เอง — ทั้ง Backend และ Langflow ไม่ต้องทำ
```

---

## กรณีพิเศษ: Backend มี Agent ของตัวเอง (ไม่ผ่าน Langflow)

เมื่อ call `POST /chat` (ไม่ใช่ `/langflow`) — Backend รัน agent ตรงๆ:

```python
# agent.py — built-in agent ใน Backend
# มี system prompt, conversation history, tools ทั้งหมดใน Python code

tools = [
    search_service.search_tool,  # ← OpenSearch KNN search
    url_ingestion_tool,           # ← fetch & index URL
    calculator_tool,              # ← math expressions
]

response = await llm_client.responses.create(
    model=config.llm_model,
    input=messages,
    tools=tools,
)
```

**เมื่อไหรใช้ `/chat` vs `/langflow`:**
| Endpoint | ใช้ | Logic อยู่ |
|---------|-----|-----------|
| `POST /chat` | Direct mode, no Langflow needed | Backend (agent.py) |
| `POST /langflow` | Full pipeline, customizable | Langflow flow |

ปกติ Frontend เรียก `/langflow` เป็นหลัก — Langflow mode ยืดหยุ่นกว่า ปรับ flow ได้ไม่ต้องแก้โค้ด
