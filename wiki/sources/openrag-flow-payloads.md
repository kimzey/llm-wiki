---
title: "OpenRAG — Flow Payloads & Permission Architecture"
type: source
source_file: raw/notes/openrag/docs-lean/openrag-flows-payload-permission.md
published: 2026-03-18
tags: [openrag, langflow, flows, payload, permission, jwt, headers]
related: [wiki/concepts/openrag-platform.md, wiki/concepts/langflow-visual-workflow.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source (primary)**: [[../../raw/notes/openrag/docs-lean/openrag-flows-payload-permission.md|Flow Payload & Permission Analysis]]
> **Full source (complete)**: [[../../raw/notes/openrag/docs-lean/openrag-langflow-flows-guide.md|Langflow Flows Guide]]
> **Additional**: [[../../raw/notes/openrag/docs-lean/openrag-langflow-flows-guide.md|Langflow Flows Guide]]

## สรุป

วิเคราะห์ payload ที่แต่ละ Langflow Flow รับ (headers + body) และ architecture ของ permission system

## ประเด็นสำคัญ

### 4 Flows ของ OpenRAG

| Flow | File | Flow ID | Backend API |
|------|------|---------|-------------|
| Chat RAG | `openrag_agent.json` | `1098eea1-...` | `POST /langflow` |
| Document Ingest | `ingestion_flow.json` | `5488df7c-...` | `POST /router/upload_ingest` |
| Nudges (Suggestions) | `openrag_nudges.json` | `ebc01d31-...` | `POST /nudges` |
| URL Ingest | `openrag_url_mcp.json` | `72c3d17c-...` | MCP tool (auto) |

### 2 ช่องทางส่งข้อมูลเข้า Flow

**HTTP Headers** = Global Variables — inject ค่าให้ TextInput node โดยตรง
```
X-LANGFLOW-GLOBAL-VAR-{VARIABLE_NAME}: value
```

**HTTP Body** = payload หลัก
```json
{
  "input_value": "...",
  "session_id": "...",
  "tweaks": { "NodeId": { "field": value } }
}
```

### Headers สรุปทุก Flow

| Variable | Chat | Ingest | Nudges | URL |
|----------|------|--------|--------|-----|
| `JWT` | ✅ | ✅ | ✅ | — |
| `SELECTED_EMBEDDING_MODEL` | ✅ | ✅ | ✅ | ✅ |
| `OPENRAG-QUERY-FILTER` | ✅ | — | ✅ | — |
| `OWNER` | — | ✅ | — | ✅ |
| `OWNER_NAME` / `OWNER_EMAIL` | — | ✅ | — | ✅ |
| `ALLOWED_USERS` | — | ✅ | — | — |
| `ALLOWED_GROUPS` | — | ✅ | — | — |
| `CONNECTOR_TYPE` | — | ✅ | — | ✅ |
| `FILENAME`, `MIMETYPE`, `FILESIZE` | — | ✅ | — | — |
| `DOCUMENT_ID`, `SOURCE_URL` | — | ✅ | — | — |

### Chat Flow — Body

```json
{
  "input_value": "นโยบายลาพักร้อนมีกี่วัน?",
  "input_type": "chat",
  "output_type": "chat",
  "session_id": "user_abc123",
  "stream": true,
  "tweaks": {}
}
```

### Ingest Flow — Tweaks (required)

```json
{
  "tweaks": {
    "DoclingRemote-Dp3PX": {
      "path": ["/tmp/langflow/uploads/file.pdf"]
    },
    "SplitText-QIKhg": {
      "chunk_size": 500,
      "chunk_overlap": 100,
      "separator": "\n\n"
    }
  }
}
```

### Permission Architecture

```
Langflow Flow → ❌ ไม่มี permission ในตัวเลย (ใครมี API key เรียกได้)

Permission จริง:
  ✅ Backend (FastAPI): ตรวจ JWT cookie / API Key
  ✅ Langflow: ต้องมี Langflow API key (server-to-server)
  ✅ OpenSearch: OIDC JWT validation + Document ACL filter (สำคัญที่สุด)
```

**Flow จริงของ Permission:**
1. Backend รับ request + JWT cookie
2. Backend inject JWT → `X-LANGFLOW-GLOBAL-VAR-JWT: eyJ...`
3. Langflow ส่ง JWT → OpenSearch
4. OpenSearch validate JWT (ดึง public key จาก `/auth/jwks`)
5. KNN search auto-filter: `owner == user.sub OR allowed_users contains user.sub`

**ถ้า bypass Backend → เรียก Langflow ตรงๆ:**
- ไม่ส่ง JWT → OpenSearch reject (security enabled)
- ส่ง JWT ถูก → ACL ยังทำงาน แต่ Rate limit/Audit log ถูก bypass

### Tweakable Node IDs

| Node ID | Flow | Override ได้ |
|---------|------|-------------|
| `DoclingRemote-Dp3PX` | Ingest | `path`, `do_ocr`, `ocr_engine` |
| `SplitText-QIKhg` | Ingest, URL | `chunk_size`, `chunk_overlap`, `separator` |
| `Agent-Nfw7u` | Chat | `model_name`, `system_prompt`, `temperature` |
| `EmbeddingModel-*` | All | `model`, `api_key` |
| `OpenSearchVectorStore-*` | All | `filter_expression`, `number_of_results` |

### 3 วิธีสร้าง Custom UI

| | ผ่าน Backend API | Public API v1 (Key) | Langflow โดยตรง |
|---|---|---|---|
| **Auth** | JWT Cookie | API Key | Langflow API Key |
| **ACL/Permission** | ✅ อัตโนมัติ | ✅ อัตโนมัติ | ⚠️ ต้อง handle เอง |
| **Streaming** | ✅ | ✅ | ✅ |
| **เหมาะกับ** | Web app | Backend/Script | Prototype/Internal |
| **Production Ready** | ✅ | ✅ | ❌ ไม่แนะนำ |

**วิธีที่ 1: ผ่าน Backend API (แนะนำ)**

```javascript
// 1. Chat (streaming)
const res = await fetch('http://localhost:8080/langflow', {
  method: 'POST',
  credentials: 'include',          // ← ส่ง JWT cookie
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    prompt: "นโยบายลาพักร้อนคืออะไร?",
    stream: true,
    filter_id: 'kf_hr_policies',
    limit: 10,
    scoreThreshold: 0.5
  })
})
// 2. Upload
const formData = new FormData()
formData.append('file', file)
formData.append('knowledge_filter_id', 'kf_engineering')
await fetch('http://localhost:8080/router/upload_ingest', {
  method: 'POST', credentials: 'include', body: formData
})
```

**วิธีที่ 2: Public API v1 (API Key)**

```python
requests.post(f"{BASE}/v1/chat",
  headers={"X-API-Key": API_KEY},
  json={"prompt": question, "stream": False})
```

**วิธีที่ 3: Langflow โดยตรง (Bypass Backend)**

```python
requests.post(f"{LANGFLOW}/api/v1/run/{FLOW_ID}",
  headers={"x-api-key": LANGFLOW_KEY,
           "x-langflow-global-var-jwt": "<jwt>"},
  json={"input_value": question, "session_id": "default"})
# ⚠️ ไม่มี ACL filter → user เห็นเอกสารทั้งหมด
```

### Authentication Modes

```
┌──────────────┬──────────────┬───────────────────────┐
│ No-Auth Mode │  OAuth Mode  │     API Key Mode      │
│              │              │                       │
│ anonymous@   │ Google/MSFT  │ orag_xxxxxxxx         │
│ localhost    │ OAuth 2.0    │ X-API-Key header      │
│              │ JWT Cookie   │                       │
│ dev/demo    │ production   │ server-to-server      │
└──────────────┴──────────────┴───────────────────────┘
```

**OAuth Flow:**
```
POST /auth/init → { authorization_url, connection_id }
→ Redirect to Google
→ POST /auth/callback → set httponly cookie: auth_token=<jwt>
→ JWT valid 7 วัน (RS256)
```

**สร้าง API Key:**
```bash
curl -X POST http://localhost:8080/keys \
  -H "Cookie: auth_token=<jwt>" \
  -d '{"name": "My Service"}'
# Response: { "api_key": "orag_1a2b3c4d..." }  ← แสดงครั้งเดียว
```

### Custom Flow สร้างเอง

```
1. Langflow UI → New Flow → Blank Flow
2. Components: ChatInput, ChatOutput, Agent, OpenSearchVectorStore,
   EmbeddingModel, TextInput (รับ JWT/config), Prompt Template
3. TextInput node ต้องมีชื่อตรง global var:
   JWT, SELECTED_EMBEDDING_MODEL, OPENRAG-QUERY-FILTER
4. หลังสร้าง → ได้ Flow ID → ใส่ใน .env
5. เพิ่ม endpoint ใน src/main.py
```

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/openrag-platform|OpenRAG Platform]]
- [[wiki/concepts/langflow-visual-workflow|Langflow]] — Flow architecture
