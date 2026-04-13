# OpenRAG — คู่มือ Langflow Flows ฉบับละเอียด

> ครอบคลุม: Flow ต่างๆ คืออะไร, Frontend เชื่อมอย่างไร,
> สร้าง Custom UI เองได้ยังไง, จัดการ Permission/Login อย่างไร

---

## สารบัญ

- [1. ภาพรวม Architecture](#1-ภาพรวม-architecture)
- [2. Flow ทั้ง 4 ตัวคืออะไร](#2-flow-ทั้ง-4-ตัวคืออะไร)
- [3. Frontend → Backend → Langflow เชื่อมกันอย่างไร](#3-frontend--backend--langflow-เชื่อมกันอย่างไร)
- [4. สร้าง Custom UI เอง](#4-สร้าง-custom-ui-เอง)
- [5. จัดการ Login และ Permission](#5-จัดการ-login-และ-permission)
- [6. Custom Flow เอง](#6-custom-flow-เอง)
- [7. ตัวอย่างโค้ดจริง](#7-ตัวอย่างโค้ดจริง)

---

## 1. ภาพรวม Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Request Flow ทั้งหมด                          │
│                                                                 │
│  Browser/App                                                    │
│      │                                                          │
│      │ HTTP Request (cookie: auth_token หรือ X-API-Key)         │
│      ▼                                                          │
│  ┌─────────────┐                                                │
│  │  FastAPI    │  ← Port 8000 (exposed as 8080)                 │
│  │  Backend    │  ← ตรวจสอบ JWT/API Key                         │
│  └──────┬──────┘                                                │
│         │ internal HTTP (bearer token)                          │
│         ▼                                                       │
│  ┌─────────────┐      ┌──────────────┐                          │
│  │  Langflow   │ ←──→ │  OpenSearch  │                          │
│  │  Port 7860  │      │  Port 9200   │                          │
│  └─────────────┘      └──────────────┘                          │
│         │                    │                                  │
│    Run Flows              Vector DB                             │
│    (RAG Agent,            (docs + ACL)                          │
│     Ingestion, etc.)                                            │
└─────────────────────────────────────────────────────────────────┘
```

**หลักการสำคัญ:**
- **Frontend ไม่คุยกับ Langflow โดยตรง** — ผ่าน Backend เสมอ
- Backend เป็น **proxy + auth layer** ก่อนส่งไป Langflow
- JWT token ส่งผ่าน HTTP header ไปยัง Langflow เพื่อ filter ข้อมูลตาม user

---

## 2. Flow ทั้ง 4 ตัวคืออะไร

### Flow 1: OpenRAG OpenSearch Agent Flow (Chat RAG)

| | |
|---|---|
| **ไฟล์** | `flows/openrag_agent.json` |
| **Flow ID** | `1098eea1-6649-4e1d-aed1-b77249fb8dd0` |
| **Env Var** | `LANGFLOW_CHAT_FLOW_ID` |
| **หน้าที่** | รับคำถาม → ค้นหาใน OpenSearch → ส่งให้ LLM → ตอบ |
| **ใช้งานผ่าน** | `POST /langflow` หรือ `POST /v1/chat` |

**Nodes ใน Flow:**
```
ChatInput → Agent → ChatOutput
              │
              ├── OpenSearchVectorStore (ค้นหา docs)
              ├── EmbeddingModel (embed query)
              ├── CalculatorComponent (คำนวณตัวเลข)
              ├── MCP (URL ingestion tool)
              └── Prompt Template (system prompt)
```

**ข้อมูลที่รับ:**
```json
{
  "input_value": "นโยบายลาพักร้อนมีกี่วัน?",
  "input_type": "chat",
  "output_type": "chat",
  "session_id": "user_abc123",
  "tweaks": {}
}
```

**Headers ที่ Backend ส่งไปให้ Langflow:**
```
X-LANGFLOW-GLOBAL-VAR-JWT: eyJhbGci...           ← JWT ของ user
X-LANGFLOW-GLOBAL-VAR-SELECTED_EMBEDDING_MODEL: text-embedding-3-small
X-LANGFLOW-GLOBAL-VAR-OPENRAG-QUERY-FILTER: {"filter": [...], "limit": 10}
X-LANGFLOW-API-KEY: <langflow_api_key>
```

---

### Flow 2: OpenSearch Ingestion Flow (Document Ingest)

| | |
|---|---|
| **ไฟล์** | `flows/ingestion_flow.json` |
| **Flow ID** | `5488df7c-b93f-4f87-a446-b67028bc0813` |
| **Env Var** | `LANGFLOW_INGEST_FLOW_ID` |
| **หน้าที่** | รับไฟล์ → Docling parse → Split → Embed → Index ใน OpenSearch |
| **ใช้งานผ่าน** | `POST /router/upload_ingest` หรือ `POST /v1/documents/ingest` |

**Nodes ใน Flow:**
```
DoclingRemote → ExportDoclingDocument → DataFrameOperations (x3)
                                              │
                                         SplitText (chunking)
                                              │
                                         EmbeddingModel
                                              │
                                         OpenSearchVectorStore (index)
```

**Tweaks ที่ Backend ส่งไป:**
```json
{
  "tweaks": {
    "DoclingRemote-Dp3PX": {
      "path": ["/tmp/uploaded_file.pdf"]
    }
  },
  "input_value": "Ingest files",
  "session_id": "upload_session_xyz"
}
```

---

### Flow 3: OpenRAG OpenSearch Nudges Flow (Suggestions)

| | |
|---|---|
| **ไฟล์** | `flows/openrag_nudges.json` |
| **Flow ID** | `ebc01d31-1976-46ce-a385-b0240327226c` |
| **Env Var** | `NUDGES_FLOW_ID` |
| **หน้าที่** | Generate คำถามแนะนำ (suggestion chips) จาก knowledge base |
| **ใช้งานผ่าน** | `POST /nudges` |

**ตัวอย่าง output:**
```json
{
  "nudges": [
    "นโยบายการลาป่วยของบริษัทคืออะไร?",
    "ขั้นตอนการเบิกค่ารักษาพยาบาลทำอย่างไร?",
    "วันหยุดประจำปี 2024 มีวันไหนบ้าง?"
  ]
}
```

---

### Flow 4: OpenSearch URL Ingestion Flow (Web Content)

| | |
|---|---|
| **ไฟล์** | `flows/openrag_url_mcp.json` |
| **Flow ID** | `72c3d17c-2dac-4a73-b48a-6518473d7830` |
| **Env Var** | `LANGFLOW_URL_INGEST_FLOW_ID` |
| **หน้าที่** | ดึงเนื้อหาจาก URL → parse → index ใน OpenSearch |
| **ถูกเรียกจาก** | MCP tool ภายใน Agent Flow (ตอน user ส่ง URL มาในการสนทนา) |

---

## 3. Frontend → Backend → Langflow เชื่อมกันอย่างไร

### ตัวอย่าง: User ถามคำถาม (Full Flow)

```
1. User พิมพ์: "นโยบายลาพักร้อนมีกี่วัน?"
   │
   ▼
2. Frontend ส่ง HTTP Request:
   POST http://localhost:8080/langflow
   Cookie: auth_token=eyJhbGci...
   Body: {
     "prompt": "นโยบายลาพักร้อนมีกี่วัน?",
     "stream": true,
     "filter_id": "kf_hr_policies",
     "limit": 10
   }
   │
   ▼
3. Backend (FastAPI) ทำงาน:
   a. ดึง JWT จาก cookie auth_token
   b. Validate JWT → ได้ user_id + email
   c. เพิ่ม JWT ลงใน headers
   d. เรียก Langflow

   POST http://langflow:7860/api/v1/run/1098eea1-6649-4e1d-aed1-b77249fb8dd0
   x-api-key: <langflow_api_key>
   x-langflow-global-var-jwt: eyJhbGci...
   x-langflow-global-var-selected_embedding_model: text-embedding-3-small
   x-langflow-global-var-openrag-query-filter: {"filter": [], "limit": 10}
   Body: {
     "input_value": "นโยบายลาพักร้อนมีกี่วัน?",
     "session_id": "user_abc123",
     "stream": true
   }
   │
   ▼
4. Langflow Agent Flow ทำงาน:
   a. Embed query: [0.023, -0.156, ...]
   b. KNN Search ใน OpenSearch พร้อม JWT filter:
      WHERE owner='user@company.com'
      OR 'user@company.com' IN allowed_users
      OR 'all_employees' IN allowed_groups
   c. ได้ chunks ที่เกี่ยวข้อง
   d. ส่งให้ LLM (GPT-4) พร้อม context
   e. LLM ตอบ: "พนักงานมีสิทธิ์ลาพักร้อน 10 วัน..."
   │
   ▼
5. Backend Stream Response กลับ Frontend:
   data: {"chunk": "พนักงาน"}
   data: {"chunk": "มีสิทธิ์ลา"}
   data: {"chunk": "พักร้อน 10 วัน"}
   data: {"sources": ["นโยบาย_HR_2024.pdf"]}
   data: [DONE]
   │
   ▼
6. Frontend แสดงผลแบบ streaming ทีละ token
```

---

### Sequence Diagram แบบย่อ

```
Frontend    Backend(FastAPI)    Langflow        OpenSearch
    │               │               │               │
    │──POST /langflow──►            │               │
    │    (cookie JWT)│               │               │
    │               │──validate JWT─►               │
    │               │◄──user info───│               │
    │               │               │               │
    │               │──POST /api/v1/run/{flow_id}──►│
    │               │    (headers: JWT, model, filter)│
    │               │               │──KNN search──►│
    │               │               │◄──chunks──────│
    │               │               │──LLM call     │
    │               │               │◄──response    │
    │               │◄──stream──────│               │
    │◄──SSE stream──│               │               │
```

---

## 4. สร้าง Custom UI เอง

### วิธีที่ 1: ผ่าน Backend API (แนะนำ)

เหมาะกับ: ทุก use case, รองรับ auth ครบ, access control ทำให้แล้ว

#### ขั้นตอนที่ 1: Login → ได้ JWT Cookie

```javascript
// วิธีที่ 1: No-Auth Mode (ไม่มี Google OAuth)
// → ไม่ต้อง login เลย ระบบเป็น anonymous อัตโนมัติ
// เรียก API ได้เลย

// วิธีที่ 2: OAuth Login
const loginWithGoogle = async () => {
  // Step 1: Init OAuth flow
  const res = await fetch('http://localhost:8080/auth/init', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      connector_type: 'google_drive',
      purpose: 'app_auth'
    })
  })
  const { authorization_url, connection_id } = await res.json()

  // Step 2: Redirect user ไป Google
  window.location.href = authorization_url

  // Step 3: Google redirect กลับมา callback URL ของเรา
  // Backend จัดการ set httponly cookie auth_token อัตโนมัติ
}
```

#### ขั้นตอนที่ 2: ตรวจสอบ Login Status

```javascript
const checkAuth = async () => {
  const res = await fetch('http://localhost:8080/auth/me', {
    credentials: 'include'  // ส่ง cookie ไปด้วย
  })

  if (res.status === 401) {
    // ยังไม่ได้ login
    return null
  }

  const user = await res.json()
  // { user_id: "108...", email: "user@company.com", name: "John Doe" }
  return user
}
```

#### ขั้นตอนที่ 3: Chat กับ Langflow Flow

```javascript
// Non-streaming
const chat = async (prompt) => {
  const res = await fetch('http://localhost:8080/langflow', {
    method: 'POST',
    credentials: 'include',          // ← ส่ง cookie JWT
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      prompt: prompt,
      stream: false,
      filter_id: 'kf_hr_policies',   // optional: จำกัด context
      limit: 10,
      scoreThreshold: 0.5
    })
  })

  const data = await res.json()
  // {
  //   response: "พนักงานมีสิทธิ์ลาพักร้อน 10 วัน...",
  //   response_id: "resp_abc123",
  //   sources: [{ filename: "นโยบาย_HR.pdf", page: 3 }]
  // }
  return data
}

// Streaming (SSE)
const chatStream = async (prompt, onChunk) => {
  const res = await fetch('http://localhost:8080/langflow', {
    method: 'POST',
    credentials: 'include',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ prompt, stream: true })
  })

  const reader = res.body.getReader()
  const decoder = new TextDecoder()

  while (true) {
    const { done, value } = await reader.read()
    if (done) break

    const chunk = decoder.decode(value)
    const lines = chunk.split('\n')

    for (const line of lines) {
      if (line.startsWith('data: ')) {
        const data = line.slice(6)
        if (data === '[DONE]') break
        try {
          const parsed = JSON.parse(data)
          onChunk(parsed)
        } catch (e) {}
      }
    }
  }
}
```

#### ขั้นตอนที่ 4: Upload เอกสาร

```javascript
const uploadDocument = async (file) => {
  const formData = new FormData()
  formData.append('file', file)
  // optional: กำหนด knowledge filter
  formData.append('knowledge_filter_id', 'kf_engineering')

  const res = await fetch('http://localhost:8080/router/upload_ingest', {
    method: 'POST',
    credentials: 'include',
    body: formData
  })

  const { task_id } = await res.json()

  // Poll task status
  const checkStatus = async () => {
    const statusRes = await fetch(`http://localhost:8080/tasks/${task_id}`, {
      credentials: 'include'
    })
    return statusRes.json()
  }

  return { task_id, checkStatus }
}
```

---

### วิธีที่ 2: ผ่าน Public API v1 (ใช้ API Key)

เหมาะกับ: backend-to-backend integration, CI/CD, script automation

```python
import requests

API_KEY = "orag_abc123xxxxxxxxxxxxxxxx"
BASE_URL = "http://localhost:8080"

# Chat
def ask(question: str) -> dict:
    res = requests.post(
        f"{BASE_URL}/v1/chat",
        headers={
            "X-API-Key": API_KEY,
            "Content-Type": "application/json"
        },
        json={
            "prompt": question,
            "stream": False
        }
    )
    return res.json()

response = ask("นโยบายลาพักร้อนมีกี่วัน?")
print(response["response"])

# Ingest
def ingest_file(file_path: str) -> dict:
    with open(file_path, 'rb') as f:
        res = requests.post(
            f"{BASE_URL}/v1/documents/ingest",
            headers={"X-API-Key": API_KEY},
            files={"file": (file_path, f)}
        )
    task = res.json()

    # Poll until done
    import time
    while True:
        status_res = requests.get(
            f"{BASE_URL}/v1/tasks/{task['task_id']}",
            headers={"X-API-Key": API_KEY}
        )
        status = status_res.json()
        if status["status"] in ("completed", "failed"):
            return status
        time.sleep(2)
```

---

### วิธีที่ 3: เรียก Langflow โดยตรง (Bypass Backend)

เหมาะกับ: prototype, internal tools ที่ไม่ต้องการ user auth

**ข้อควรระวัง:** ไม่มี ACL filter → user เห็นเอกสารทั้งหมด

```python
import requests

LANGFLOW_URL = "http://localhost:7860"
LANGFLOW_API_KEY = "sk-xxxxxxxxxxxxxxxxxxxxxxxx"  # จาก Langflow admin
CHAT_FLOW_ID = "1098eea1-6649-4e1d-aed1-b77249fb8dd0"

def chat_direct(question: str, session_id: str = "default") -> str:
    res = requests.post(
        f"{LANGFLOW_URL}/api/v1/run/{CHAT_FLOW_ID}",
        headers={
            "x-api-key": LANGFLOW_API_KEY,
            "Content-Type": "application/json",
            # ถ้าต้องการ JWT filter (ต้องสร้าง JWT เอง)
            "x-langflow-global-var-jwt": "<your_jwt_here>"
        },
        json={
            "input_value": question,
            "input_type": "chat",
            "output_type": "chat",
            "session_id": session_id
        }
    )
    data = res.json()
    # Extract response text
    return data["outputs"][0]["outputs"][0]["results"]["message"]["text"]

print(chat_direct("Hello, what can you help me with?"))
```

**ดู Langflow API Docs ได้ที่:** `http://localhost:7860/docs`

---

## 5. จัดการ Login และ Permission

### Authentication Flow ทั้งหมด

```
┌─────────────────────────────────────────────────────┐
│               Authentication Options                 │
├──────────────┬──────────────┬───────────────────────┤
│ No-Auth Mode │  OAuth Mode  │     API Key Mode      │
│              │              │                       │
│ anonymous@   │ Google/MSFT  │ orag_xxxxxxxx         │
│ localhost    │ OAuth 2.0    │ X-API-Key header      │
│              │ JWT Cookie   │                       │
│ เหมาะกับ:   │ เหมาะกับ:   │ เหมาะกับ:            │
│ dev/demo    │ production   │ server-to-server      │
└──────────────┴──────────────┴───────────────────────┘
```

---

### Mode 1: No-Auth Mode (สำหรับ dev/internal)

```env
# .env — ไม่ใส่ Google OAuth credentials
# GOOGLE_OAUTH_CLIENT_ID=
# GOOGLE_OAUTH_CLIENT_SECRET=
```

ระบบจะ: สร้าง `anonymous@localhost` user อัตโนมัติ, ไม่ต้อง login

```javascript
// ไม่ต้องทำ login step เลย — เรียก API ได้ทันที
const res = await fetch('/langflow', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ prompt: 'hello' })
  // ไม่ต้องใส่ credentials: 'include' เพราะไม่มี cookie
})
```

---

### Mode 2: OAuth Login (สำหรับ production)

**ขั้นตอนทั้งหมด:**

```
1. User กด "Login with Google"
         │
         ▼
2. POST /auth/init → { authorization_url, connection_id }
         │
         ▼
3. Redirect ไป Google: https://accounts.google.com/o/oauth2/...
         │
         ▼
4. Google redirect กลับมา callback URL ของ app
         │
         ▼
5. Frontend เรียก POST /auth/callback:
   { connection_id, authorization_code, state }
         │
         ▼
6. Backend:
   - ส่ง code ไป Google → ได้ access_token + user_info
   - สร้าง JWT (RS256, 7 วัน)
   - Set httponly cookie: auth_token=<jwt>
         │
         ▼
7. User authenticated แล้ว
   ทุก request ส่ง cookie auth_token ไปอัตโนมัติ
```

**ตัวอย่าง Full Login Flow (React):**

```jsx
// LoginButton.jsx
export function LoginButton() {
  const handleLogin = async () => {
    // Step 1: Init OAuth
    const res = await fetch('/auth/init', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        connector_type: 'google_drive',
        purpose: 'app_auth',
        redirect_uri: 'http://localhost:3000/auth/callback'
      })
    })
    const { authorization_url } = await res.json()

    // Step 2: Redirect ไป Google
    window.location.href = authorization_url
  }

  return <button onClick={handleLogin}>Login with Google</button>
}

// CallbackPage.jsx (ที่ /auth/callback)
export function CallbackPage() {
  useEffect(() => {
    const params = new URLSearchParams(window.location.search)
    const code = params.get('code')
    const state = params.get('state')
    const connectionId = localStorage.getItem('connection_id')

    // Step 3: Exchange code for JWT
    fetch('/auth/callback', {
      method: 'POST',
      credentials: 'include',  // ← สำคัญ: รับ cookie กลับมา
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        connection_id: connectionId,
        authorization_code: code,
        state: state
      })
    }).then(() => {
      window.location.href = '/'  // กลับหน้าหลัก
    })
  }, [])

  return <div>Logging in...</div>
}
```

---

### Mode 3: API Key Auth (สำหรับ server-to-server)

**สร้าง API Key:**

```bash
# ต้อง login ก่อน (ผ่าน OAuth) แล้วค่อยสร้าง key
curl -X POST http://localhost:8080/keys \
  -H "Cookie: auth_token=<your_jwt>" \
  -H "Content-Type: application/json" \
  -d '{"name": "My Service Integration"}'

# Response:
# {
#   "key_id": "key_xyz789",
#   "api_key": "orag_1a2b3c4d5e6f...",  ← เก็บไว้! แสดงครั้งเดียว
#   "prefix": "orag_1a2b"
# }
```

**ใช้ API Key:**

```bash
# ใน header
curl -X POST http://localhost:8080/v1/chat \
  -H "X-API-Key: orag_1a2b3c4d5e6f..." \
  -H "Content-Type: application/json" \
  -d '{"prompt": "hello"}'

# หรือใน Authorization header
curl -X POST http://localhost:8080/v1/chat \
  -H "Authorization: Bearer orag_1a2b3c4d5e6f..." \
  -d '{"prompt": "hello"}'
```

---

### Permission / Access Control

#### การตั้ง Permission ตอน Upload

```javascript
// Upload พร้อม permission
const formData = new FormData()
formData.append('file', file)

// กำหนดว่าใครเข้าถึงได้
const metadata = {
  allowed_users: ['ceo@company.com', 'cfo@company.com'],
  allowed_groups: ['finance', 'executive']
}
formData.append('metadata', JSON.stringify(metadata))

await fetch('/router/upload_ingest', {
  method: 'POST',
  credentials: 'include',
  body: formData
})
```

#### การ Filter ผลลัพธ์ใน Chat

```javascript
// Chat พร้อม filter ว่าจะค้นหาจาก source ไหน
const res = await fetch('/langflow', {
  method: 'POST',
  credentials: 'include',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    prompt: 'นโยบายลาพักร้อนคืออะไร?',

    // Filter ต่างๆ
    filters: {
      data_sources: ['นโยบาย_HR_2024.pdf'],  // เฉพาะไฟล์นี้
      document_types: ['application/pdf'],    // เฉพาะ PDF
      owners: ['hr@company.com']              // เฉพาะของ hr
    },

    filter_id: 'kf_hr_only',  // หรือใช้ Knowledge Filter
    limit: 5,
    scoreThreshold: 0.6
  })
})
```

---

## 6. Custom Flow เอง

### ขั้นตอนสร้าง Custom Flow ใน Langflow

**1. เข้า Langflow UI:**
```
http://localhost:7860
username: admin
password: <LANGFLOW_SUPERUSER_PASSWORD>
```

**2. สร้าง Flow ใหม่:**
```
New Flow → Blank Flow
หรือ Import จาก flows/*.json เป็น template
```

**3. Components ที่สำคัญ:**

```
ChatInput        ← รับ user input
ChatOutput       ← ส่ง response กลับ
Agent            ← LLM agent พร้อม tools
OpenSearchVectorStoreComponentMultimodalMultiEmbedding ← ค้นหา docs
EmbeddingModel   ← สร้าง embeddings (OpenAI/Anthropic/Ollama)
TextInput        ← รับ global variables (JWT, model, filters)
Prompt Template  ← custom system prompt
```

**4. รับ Global Variables (JWT + Config) ใน Custom Flow:**

ใน Flow ของเราต้องมี `TextInput` node ที่ชื่อตรงกับ global var:

```
Node Type: TextInput
Variable Name: JWT         ← รับ JWT token จาก backend
Variable Name: SELECTED_EMBEDDING_MODEL
Variable Name: OPENRAG-QUERY-FILTER
```

Backend ส่งผ่าน header:
```
X-LANGFLOW-GLOBAL-VAR-JWT: <token>
X-LANGFLOW-GLOBAL-VAR-SELECTED_EMBEDDING_MODEL: text-embedding-3-small
```

**5. Register Flow ใน Backend:**

หลังสร้าง Flow ใน Langflow UI → ได้ Flow ID → ใส่ใน `.env`:

```env
# เพิ่ม flow id ใหม่
LANGFLOW_CHAT_FLOW_ID=<your-new-flow-id>
# หรือเพิ่ม custom env var สำหรับ flow ใหม่
MY_CUSTOM_FLOW_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

**6. เพิ่ม Endpoint ใน Backend:**

สร้างไฟล์ `src/api/my_custom_flow.py`:

```python
from fastapi import Depends
from fastapi.responses import JSONResponse, StreamingResponse
from dependencies import get_current_user, get_chat_service
from session_manager import User
import os

MY_FLOW_ID = os.getenv("MY_CUSTOM_FLOW_ID")

async def my_custom_endpoint(
    body: dict,
    user: User = Depends(get_current_user),
):
    langflow_client = ...  # ดูตัวอย่างจาก chat_service.py

    result = await langflow_client.run_flow(
        flow_id=MY_FLOW_ID,
        input_value=body["prompt"],
        session_id=user.user_id,
        extra_headers={
            "X-LANGFLOW-GLOBAL-VAR-JWT": user.jwt_token
        }
    )
    return JSONResponse(result)
```

เพิ่มใน `src/main.py`:

```python
app.add_api_route("/my-custom", my_custom_endpoint, methods=["POST"])
```

---

### ตัวอย่าง Custom Flow: Summarization Bot

สถานการณ์: ต้องการ bot ที่ **สรุปเอกสาร** แทนที่จะตอบแบบ Q&A

```
Flow Design:
  ChatInput (รับชื่อไฟล์)
      │
  OpenSearchVectorStore (ดึงทุก chunk ของไฟล์นั้น)
      │
  Prompt Template (system: "summarize the following...")
      │
  LLM (Claude/GPT)
      │
  ChatOutput
```

เรียกใช้ผ่าน API:

```bash
curl -X POST http://localhost:8080/summarize \
  -H "Cookie: auth_token=<jwt>" \
  -H "Content-Type: application/json" \
  -d '{
    "filename": "annual_report_2024.pdf",
    "style": "bullet_points"
  }'
```

---

## 7. ตัวอย่างโค้ดจริง

### Example 1: React Chat App (ครบวงจร)

```jsx
// ChatApp.jsx
import { useState, useEffect } from 'react'

const API = 'http://localhost:8080'

export default function ChatApp() {
  const [user, setUser] = useState(null)
  const [messages, setMessages] = useState([])
  const [input, setInput] = useState('')
  const [loading, setLoading] = useState(false)
  const [responseId, setResponseId] = useState(null)

  // Check auth on mount
  useEffect(() => {
    fetch(`${API}/auth/me`, { credentials: 'include' })
      .then(r => r.json())
      .then(data => {
        if (data.user_id) setUser(data)
      })
  }, [])

  const sendMessage = async () => {
    if (!input.trim()) return

    const userMsg = { role: 'user', content: input }
    setMessages(prev => [...prev, userMsg])
    setInput('')
    setLoading(true)

    // Streaming response
    const res = await fetch(`${API}/langflow`, {
      method: 'POST',
      credentials: 'include',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        prompt: input,
        stream: true,
        previous_response_id: responseId,  // ← เก็บ conversation context
        limit: 10,
        scoreThreshold: 0.5
      })
    })

    let fullResponse = ''
    const assistantMsg = { role: 'assistant', content: '' }
    setMessages(prev => [...prev, assistantMsg])

    const reader = res.body.getReader()
    const decoder = new TextDecoder()

    while (true) {
      const { done, value } = await reader.read()
      if (done) break

      const text = decoder.decode(value)
      for (const line of text.split('\n')) {
        if (!line.startsWith('data: ')) continue
        const data = line.slice(6)
        if (data === '[DONE]') continue

        try {
          const parsed = JSON.parse(data)
          if (parsed.chunk) {
            fullResponse += parsed.chunk
            setMessages(prev => {
              const updated = [...prev]
              updated[updated.length - 1].content = fullResponse
              return updated
            })
          }
          if (parsed.response_id) {
            setResponseId(parsed.response_id)
          }
        } catch {}
      }
    }

    setLoading(false)
  }

  if (!user) {
    return (
      <div>
        <h1>Please Login</h1>
        <button onClick={() => {
          fetch(`${API}/auth/init`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
              connector_type: 'google_drive',
              purpose: 'app_auth'
            })
          }).then(r => r.json()).then(({ authorization_url }) => {
            window.location.href = authorization_url
          })
        }}>
          Login with Google
        </button>
      </div>
    )
  }

  return (
    <div>
      <h1>Welcome, {user.name}</h1>

      <div className="messages">
        {messages.map((msg, i) => (
          <div key={i} className={msg.role}>
            <strong>{msg.role}:</strong> {msg.content}
          </div>
        ))}
        {loading && <div>...</div>}
      </div>

      <input
        value={input}
        onChange={e => setInput(e.target.value)}
        onKeyDown={e => e.key === 'Enter' && sendMessage()}
        placeholder="ถามคำถาม..."
      />
      <button onClick={sendMessage} disabled={loading}>Send</button>
    </div>
  )
}
```

---

### Example 2: Python Integration Script

```python
"""
ตัวอย่าง: Script ที่ ingest เอกสารทุกคืน แล้ว query ผ่าน API Key
"""
import requests
import time
import os
from pathlib import Path

API_KEY = os.environ["OPENRAG_API_KEY"]  # orag_xxx
BASE = "http://localhost:8080"

def ingest_folder(folder: str) -> list[str]:
    """Ingest ไฟล์ทั้งหมดใน folder"""
    task_ids = []
    for filepath in Path(folder).glob("*.pdf"):
        print(f"Ingesting: {filepath.name}")
        with open(filepath, 'rb') as f:
            res = requests.post(
                f"{BASE}/v1/documents/ingest",
                headers={"X-API-Key": API_KEY},
                files={"file": (filepath.name, f, "application/pdf")}
            )
        task_ids.append(res.json()["task_id"])
    return task_ids

def wait_for_tasks(task_ids: list[str]) -> None:
    """รอให้ task ทั้งหมดเสร็จ"""
    for task_id in task_ids:
        while True:
            res = requests.get(
                f"{BASE}/v1/tasks/{task_id}",
                headers={"X-API-Key": API_KEY}
            )
            status = res.json()["status"]
            if status == "completed":
                print(f"Task {task_id}: ✓")
                break
            elif status == "failed":
                print(f"Task {task_id}: ✗ FAILED")
                break
            time.sleep(2)

def ask(question: str) -> dict:
    """ถามคำถาม"""
    res = requests.post(
        f"{BASE}/v1/chat",
        headers={
            "X-API-Key": API_KEY,
            "Content-Type": "application/json"
        },
        json={"prompt": question, "stream": False}
    )
    return res.json()

# Usage
if __name__ == "__main__":
    # 1. Ingest เอกสารใหม่
    tasks = ingest_folder("/data/new_documents")
    wait_for_tasks(tasks)

    # 2. Query
    result = ask("สรุปนโยบายใหม่ที่เพิ่มมาล่าสุด")
    print(result["response"])
    print("Sources:", [s["filename"] for s in result.get("sources", [])])
```

---

### Example 3: ตัวอย่าง Response จริงจาก API

**Chat Response (non-stream):**
```json
{
  "response": "จากเอกสารนโยบาย HR พนักงานมีสิทธิ์ลาพักร้อนประจำปีไม่เกิน 10 วันทำการ โดยต้องแจ้งล่วงหน้าอย่างน้อย 3 วัน และได้รับการอนุมัติจากหัวหน้างาน (Source: นโยบาย_HR_2024.pdf, หน้า 3)",
  "response_id": "resp_7f3a9b2c",
  "sources": [
    {
      "filename": "นโยบาย_HR_2024.pdf",
      "page": 3,
      "score": 0.94,
      "text": "พนักงานมีสิทธิ์ลาพักร้อนประจำปีไม่เกิน 10 วันทำการ..."
    }
  ]
}
```

**Chat Stream (SSE):**
```
data: {"chunk": "จาก"}
data: {"chunk": "เอกสาร"}
data: {"chunk": "นโยบาย HR"}
data: {"chunk": " พนักงาน"}
data: {"chunk": "มีสิทธิ์ลา"}
...
data: {"response_id": "resp_7f3a9b2c"}
data: {"sources": [{"filename": "นโยบาย_HR_2024.pdf", "page": 3}]}
data: [DONE]
```

**Auth Me Response:**
```json
{
  "user_id": "108234567890",
  "email": "john.doe@company.com",
  "name": "John Doe",
  "picture": "https://lh3.googleusercontent.com/...",
  "provider": "google",
  "authenticated": true
}
```

---

## สรุปเปรียบเทียบ 3 วิธีสร้าง Custom UI

| | ผ่าน Backend API | Public API v1 (Key) | Langflow โดยตรง |
|---|---|---|---|
| **Auth** | JWT Cookie | API Key | Langflow Key |
| **ACL/Permission** | ✅ อัตโนมัติ | ✅ อัตโนมัติ | ⚠️ ต้อง handle เอง |
| **Streaming** | ✅ | ✅ | ✅ |
| **เหมาะกับ** | Web app | Backend/Script | Prototype/Internal |
| **ความยาก** | ปานกลาง | ง่าย | ง่าย (แต่ risk สูง) |
| **Production Ready** | ✅ | ✅ | ❌ ไม่แนะนำ |

---

*อ้างอิง source code:*
- `src/services/chat_service.py` — langflow chat integration
- `src/api/chat.py` — chat endpoints
- `src/api/auth.py` — OAuth endpoints
- `src/dependencies.py` — auth middleware
- `flows/*.json` — flow definitions
