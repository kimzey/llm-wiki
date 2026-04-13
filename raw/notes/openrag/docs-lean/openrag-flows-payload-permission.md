# OpenRAG — Flow Payload & Permission Analysis (ฉบับละเอียด)

> **คำถามหลัก 2 ข้อ:**
> 1. แต่ละ Flow รับข้อมูล/payload อะไรเข้าไปบ้าง?
> 2. Flow มีระบบ Permission ในตัวหรือเปล่า? ใครก็ตามสามารถเรียกได้ไหม?

---

## สารบัญ

- [ภาพรวม: 2 ช่องทางส่งข้อมูลเข้า Flow](#ภาพรวม-2-ช่องทางส่งข้อมูลเข้า-flow)
- [Flow 1: openrag_agent (Chat RAG)](#flow-1-openrag_agent-chat-rag)
- [Flow 2: ingestion_flow (Document Ingest)](#flow-2-ingestion_flow-document-ingest)
- [Flow 3: openrag_nudges (Suggestions)](#flow-3-openrag_nudges-suggestions)
- [Flow 4: openrag_url_mcp (URL Ingest)](#flow-4-openrag_url_mcp-url-ingest)
- [Permission: Flow มีหรือไม่มี?](#permission-flow-มีหรือไม่มี)
- [ตารางสรุป Payload ทั้งหมด](#ตารางสรุป-payload-ทั้งหมด)

---

## ภาพรวม: 2 ช่องทางส่งข้อมูลเข้า Flow

Langflow รับข้อมูลผ่าน **2 ช่องทาง** พร้อมกัน:

```
POST /api/v1/run/{flow_id}
├── HTTP Headers  ← "Global Variables" — ส่งค่าไปให้ TextInput node โดยตรง
│   X-LANGFLOW-GLOBAL-VAR-{VARIABLE_NAME}: value
│
└── HTTP Body (JSON) ← payload หลัก
    {
      "input_value": "...",     ← เข้า ChatInput node
      "session_id": "...",      ← ระบุ conversation context
      "tweaks": {               ← override ค่าใน node ใดก็ได้
        "NodeId-xxxxx": {
          "field_name": value
        }
      }
    }
```

**หลักการ:**
- **Headers** = ข้อมูล metadata, credentials, permission — Backend inject ก่อนส่งไป Langflow
- **Body `input_value`** = ข้อความจาก user / input หลัก — ไหลเข้า `ChatInput` node
- **Body `tweaks`** = override configuration ของ node ใดก็ได้ใน flow ณ runtime

---

## Flow 1: openrag_agent (Chat RAG)

**ไฟล์:** `flows/openrag_agent.json`
**Flow ID:** `1098eea1-6649-4e1d-aed1-b77249fb8dd0`
**Env:** `LANGFLOW_CHAT_FLOW_ID`
**เรียกผ่าน Backend:** `POST /langflow`

### Node Map

```
TextInput-i0a3P  ──────────────────────────────────────────┐
[Variable: (empty)]                                         │ filter_expression
                                                            ▼
ChatInput-ci8VE ──► Prompt Template-7kZsI ──► Agent-Nfw7u ──► ChatOutput-gWl8E
[user message]                                 │
                                               ├── OpenSearchVectorStore-TyvvE ◄── EmbeddingModel (x3)
                                               ├── CalculatorComponent-KrlMH
                                               └── MCP-7EY21 (URL ingestion tool)
```

### HTTP Headers ที่ Backend ส่งไป

```
X-LANGFLOW-GLOBAL-VAR-JWT: eyJhbGci...
    └─► TextInput node → OpenSearchVectorStore (OIDC auth filter)
    ค่า: JWT token ของ user จาก cookie auth_token

X-LANGFLOW-GLOBAL-VAR-SELECTED_EMBEDDING_MODEL: text-embedding-3-small
    └─► EmbeddingModel nodes (3 ตัว)
    ค่า: ชื่อ model จาก config.yaml

X-LANGFLOW-GLOBAL-VAR-OPENRAG-QUERY-FILTER: {"filter": [...], "limit": 10}
    └─► TextInput-i0a3P → OpenSearchVectorStore filter_expression
    ค่า: JSON string ของ filter
```

### HTTP Body

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

### Tweaks ที่รองรับ (override node ได้)

```json
{
  "tweaks": {
    "Agent-Nfw7u": {
      "model_name": "gpt-4o",
      "system_prompt": "You are a Thai assistant..."
    },
    "OpenSearchVectorStoreComponentMultimodalMultiEmbedding-TyvvE": {
      "filter_expression": "{\"filter\": [{\"term\": {\"filename\": \"policy.pdf\"}}]}",
      "number_of_results": 5
    },
    "EmbeddingModel-aIP4U": {
      "model": "text-embedding-3-large"
    }
  }
}
```

### Response ที่ได้กลับมา

```json
{
  "session_id": "user_abc123",
  "outputs": [
    {
      "outputs": [
        {
          "results": {
            "message": {
              "text": "พนักงานมีสิทธิ์ลาพักร้อน 10 วันทำการ (Source: นโยบาย_HR.pdf)",
              "sender": "Machine",
              "session_id": "user_abc123"
            }
          }
        }
      ]
    }
  ]
}
```

---

## Flow 2: ingestion_flow (Document Ingest)

**ไฟล์:** `flows/ingestion_flow.json`
**Flow ID:** `5488df7c-b93f-4f87-a446-b67028bc0813`
**Env:** `LANGFLOW_INGEST_FLOW_ID`
**เรียกผ่าน Backend:** `POST /router/upload_ingest`

### Node Map

```
TextInput nodes (8 ตัว) ──► AdvancedDynamicFormBuilder-81Exw
  OWNER                                │
  OWNER_NAME                           │ docs_metadata (ACL + ownership)
  OWNER_EMAIL                          ▼
  ALLOWED_USERS          DoclingRemote-Dp3PX (parse file)
  ALLOWED_GROUPS              │
  CONNECTOR_TYPE              ▼
  DOCUMENT_ID            ExportDoclingDocument
  SOURCE_URL                  │
                              ▼
                         DataFrameOperations (x3)
                              │
                              ▼
                         SplitText-QIKhg (chunk)
                              │
                              ▼
                         OpenSearchVectorStore ◄── EmbeddingModel (x3)
                         [index + ACL metadata]
```

### HTTP Headers ที่ Backend ส่งไป (ทั้งหมด)

```
X-Langflow-Global-Var-JWT: eyJhbGci...
    ค่า: JWT ของ user ที่ upload

X-Langflow-Global-Var-OWNER: user@company.com
    ค่า: user_id ของผู้ upload (email หรือ sub claim)

X-Langflow-Global-Var-OWNER_NAME: John Doe
    ค่า: ชื่อผู้ upload

X-Langflow-Global-Var-OWNER_EMAIL: user@company.com
    ค่า: email ผู้ upload

X-Langflow-Global-Var-CONNECTOR_TYPE: upload
    ค่า: "upload", "google_drive", "onedrive", "sharepoint", "s3"

X-Langflow-Global-Var-FILENAME: นโยบาย_HR_2024.pdf
    ค่า: ชื่อไฟล์จริง

X-Langflow-Global-Var-MIMETYPE: application/pdf
    ค่า: MIME type ของไฟล์

X-Langflow-Global-Var-FILESIZE: 1048576
    ค่า: ขนาดไฟล์เป็น bytes

X-Langflow-Global-Var-SELECTED_EMBEDDING_MODEL: text-embedding-3-small
    ค่า: embedding model ที่ใช้ index

X-Langflow-Global-Var-DOCUMENT_ID: doc_abc123
    ค่า: ID ของเอกสาร (จาก connector หรือ hash)

X-Langflow-Global-Var-SOURCE_URL: https://drive.google.com/...
    ค่า: URL ต้นทาง (ถ้ามาจาก connector)

X-Langflow-Global-Var-ALLOWED_USERS: ["ceo@company.com"]
    ค่า: JSON array ของ user ที่มีสิทธิ์เข้าถึง

X-Langflow-Global-Var-ALLOWED_GROUPS: ["hr", "all_employees"]
    ค่า: JSON array ของ group ที่มีสิทธิ์เข้าถึง
```

### HTTP Body

```json
{
  "input_value": "Ingest files",
  "input_type": "chat",
  "output_type": "text",
  "session_id": "upload_session_xyz",
  "tweaks": {
    "DoclingRemote-Dp3PX": {
      "path": ["/tmp/langflow/uploads/นโยบาย_HR_2024.pdf"]
    }
  }
}
```

### Tweaks ที่รองรับ (override ได้)

```json
{
  "tweaks": {
    "DoclingRemote-Dp3PX": {
      "path": ["/tmp/file.pdf"],
      "do_ocr": true,
      "ocr_engine": "easyocr"
    },
    "SplitText-QIKhg": {
      "chunk_size": 500,
      "chunk_overlap": 100,
      "separator": "\n\n"
    },
    "EmbeddingModel-EAo9i": {
      "model": "text-embedding-3-large"
    }
  }
}
```

### ข้อมูลที่เก็บใน OpenSearch หลัง Ingest

```json
{
  "document_id": "doc_abc123",
  "filename": "นโยบาย_HR_2024.pdf",
  "mimetype": "application/pdf",
  "page": 3,
  "text": "พนักงานมีสิทธิ์ลาพักร้อนประจำปีไม่เกิน 10 วัน...",
  "chunk_embedding_text-embedding-3-small": [0.023, -0.156, ...],
  "embedding_model": "text-embedding-3-small",
  "owner": "user@company.com",
  "owner_name": "John Doe",
  "owner_email": "user@company.com",
  "allowed_users": ["ceo@company.com"],
  "allowed_groups": ["hr", "all_employees"],
  "connector_type": "upload",
  "source_url": "",
  "indexed_time": "2024-01-15T09:30:00Z"
}
```

---

## Flow 3: openrag_nudges (Suggestions)

**ไฟล์:** `flows/openrag_nudges.json`
**Flow ID:** `ebc01d31-1976-46ce-a385-b0240327226c`
**Env:** `NUDGES_FLOW_ID`
**เรียกผ่าน Backend:** `POST /nudges`

### Node Map

```
TextInput-BG2U3 ──────────────────────────────────────────────────┐
[Variable: OPENRAG-QUERY-FILTER]                                   │ filter_expression
                                                                   ▼
ChatInput-7W1BE ──► Prompt Template-Wo6kR ◄── OpenSearchVectorStore-0ByE3 ◄── EmbeddingModel (x3)
                          │                    [ค้นหา docs เพื่อ generate nudges]
                          ▼
                    LanguageModelComponent (LLM)
                          │
                          ▼
                    ChatOutput-axewE
```

### HTTP Headers ที่ Backend ส่งไป

```
X-LANGFLOW-GLOBAL-VAR-JWT: eyJhbGci...
    ค่า: JWT ของ user (filter documents ตาม ACL)

X-LANGFLOW-GLOBAL-VAR-SELECTED_EMBEDDING_MODEL: text-embedding-3-small

X-LANGFLOW-GLOBAL-VAR-OPENRAG-QUERY-FILTER: {"filter": [], "limit": 5}
    └─► TextInput-BG2U3 → OpenSearchVectorStore filter_expression
```

### HTTP Body

```json
{
  "input_value": "",
  "input_type": "chat",
  "output_type": "chat",
  "session_id": "nudges_session"
}
```

### Response ตัวอย่าง

```json
{
  "outputs": [{
    "outputs": [{
      "results": {
        "message": {
          "text": "1. นโยบายการลาพักร้อนประจำปีคืออะไร?\n2. ขั้นตอนการเบิกค่ารักษาพยาบาลทำอย่างไร?\n3. วันหยุดนักขัตฤกษ์ปี 2024 มีวันอะไรบ้าง?"
        }
      }
    }]
  }]
}
```

---

## Flow 4: openrag_url_mcp (URL Ingest)

**ไฟล์:** `flows/openrag_url_mcp.json`
**Flow ID:** `72c3d17c-2dac-4a73-b48a-6518473d7830`
**Env:** `LANGFLOW_URL_INGEST_FLOW_ID`
**เรียกจาก:** MCP tool ภายใน Agent Flow (ตอน user ส่ง URL ในบทสนทนา)

### Node Map

```
TextInput nodes (4 ตัว) ──► AdvancedDynamicFormBuilder
  OWNER                              │
  OWNER_NAME                         │ docs_metadata
  OWNER_EMAIL                        ▼
  CONNECTOR_TYPE         ChatInput (URL) ──► URLComponent ──► DataFrameOperations (x4)
                                                                    │
                                                               SplitText (chunk)
                                                                    │
                                                              OpenSearchVectorStore ◄── EmbeddingModel (x3)
```

### HTTP Headers ที่ Backend ส่งไป

```
X-LANGFLOW-GLOBAL-VAR-OWNER: user@company.com
X-LANGFLOW-GLOBAL-VAR-OWNER_NAME: John Doe
X-LANGFLOW-GLOBAL-VAR-OWNER_EMAIL: user@company.com
X-LANGFLOW-GLOBAL-VAR-CONNECTOR_TYPE: url
X-LANGFLOW-GLOBAL-VAR-SELECTED_EMBEDDING_MODEL: text-embedding-3-small
```

### HTTP Body

```json
{
  "input_value": "https://docs.company.com/policy",
  "input_type": "chat",
  "output_type": "chat",
  "session_id": "url_ingest_xyz"
}
```

### การเรียกผ่าน MCP tool ใน Agent (อัตโนมัติ)

เมื่อ user พิมพ์ใน chat ว่า: `"ช่วย ingest เนื้อหาจาก https://docs.company.com ด้วย"`

Agent flow จะ detect ว่าเป็น URL → เรียก MCP tool → trigger URL ingestion flow อัตโนมัติ

---

## Permission: Flow มีหรือไม่มี?

### คำตอบ: **Flow เองไม่มี permission — แต่ระบบมี permission ผ่าน OpenSearch + JWT**

```
┌─────────────────────────────────────────────────────────────┐
│                  Permission Architecture                     │
│                                                             │
│  ❌ Langflow Flow ── ไม่มี permission checking ในตัวเลย     │
│                     ใครมี API key ก็เรียกได้ทั้งนั้น        │
│                                                             │
│  ✅ Permission จริงอยู่ที่ OpenSearch + JWT                  │
│     ├── JWT token → OpenSearch OIDC validation              │
│     ├── Document ACL fields: owner, allowed_users, groups   │
│     └── KNN search auto-filter ตาม user identity           │
└─────────────────────────────────────────────────────────────┘
```

### กลไก Permission จริงๆ ทำงานอย่างไร

```
1. Backend รับ request + JWT cookie จาก user
         │
         ▼
2. Backend inject JWT ลงใน header:
   X-LANGFLOW-GLOBAL-VAR-JWT: eyJhbGci...
         │
         ▼
3. Langflow ส่ง JWT ต่อไปยัง OpenSearch component
         │
         ▼
4. OpenSearch validate JWT ผ่าน OIDC:
   GET http://backend:8000/auth/jwks → public key
   → verify JWT signature
   → extract user_id จาก sub claim
         │
         ▼
5. OpenSearch KNN search กรองอัตโนมัติ:
   filter: {
     bool: {
       should: [
         { term: { owner: "user@company.com" } },
         { term: { allowed_users: "user@company.com" } },
         { term: { allowed_groups: "engineering" } }
       ]
     }
   }
         │
         ▼
6. ผลลัพธ์ที่ได้ = เฉพาะ docs ที่ user มีสิทธิ์
```

### ถ้าเรียก Langflow โดยตรง (bypass Backend) — เกิดอะไร?

```bash
# ❌ เรียกตรงโดยไม่ส่ง JWT
curl -X POST http://localhost:7860/api/v1/run/1098eea1-... \
  -H "x-api-key: <langflow_key>" \
  -d '{"input_value": "what documents do you have?"}'
```

**ผลที่เกิด:**

```
Case 1: ไม่ส่ง JWT header เลย
  → OpenSearch ไม่มี OIDC token → ขึ้นอยู่กับ OpenSearch security config
  → ถ้า security enabled → reject query
  → ถ้า security disabled → เห็น documents ทั้งหมด ⚠️ DANGEROUS

Case 2: ส่ง JWT ของ user A ปลอมไปกับ user B
  → OpenSearch validate JWT → signature ไม่ตรง → reject
  → ปลอม JWT ไม่ได้ (ต้องมี private key ของ backend)

Case 3: ส่ง JWT ถูกต้อง แต่ bypass Backend
  → OpenSearch filter ทำงานตามปกติ ✓ ACL ยังทำงาน
  → แต่ Backend-level validation (rate limit, audit log) ถูก bypass
```

### สรุป: ชั้นป้องกัน

```
ชั้นที่ 1: Backend API (FastAPI)
  ✅ ตรวจสอบ JWT cookie / API Key
  ✅ Rate limiting (ถ้า config)
  ✅ Audit logging
  ❌ ถ้า bypass ไปหา Langflow ตรงๆ → ข้ามชั้นนี้ได้

ชั้นที่ 2: Langflow
  ❌ ไม่มี user-level permission
  ✅ ต้องมี Langflow API key (server-to-server protection)
  ✅ Flow logic กำหนดว่าจะ search/index อะไร

ชั้นที่ 3: OpenSearch (ชั้นสำคัญที่สุด)
  ✅ OIDC JWT validation
  ✅ Document-level ACL filter
  ✅ ถ้าไม่มี JWT ที่ถูกต้อง → ไม่ได้ข้อมูล
  ✅ JWT ปลอมไม่ได้ (RSA signed)
```

---

## ตารางสรุป Payload ทั้งหมด

### Headers (Global Variables)

| Variable | Chat Flow | Ingest Flow | Nudges Flow | URL Flow | ความหมาย |
|----------|-----------|-------------|-------------|----------|----------|
| `JWT` | ✅ | ✅ | ✅ | — | JWT token สำหรับ OpenSearch ACL |
| `SELECTED_EMBEDDING_MODEL` | ✅ | ✅ | ✅ | ✅ | ชื่อ embedding model |
| `OPENRAG-QUERY-FILTER` | ✅ | — | ✅ | — | JSON filter สำหรับ search |
| `OWNER` | — | ✅ | — | ✅ | user_id ของ owner |
| `OWNER_NAME` | — | ✅ | — | ✅ | ชื่อ owner |
| `OWNER_EMAIL` | — | ✅ | — | ✅ | email owner |
| `ALLOWED_USERS` | — | ✅ | — | — | JSON array ของ user ที่เข้าถึงได้ |
| `ALLOWED_GROUPS` | — | ✅ | — | — | JSON array ของ group ที่เข้าถึงได้ |
| `CONNECTOR_TYPE` | — | ✅ | — | ✅ | ประเภท source (upload/gdrive/...) |
| `FILENAME` | — | ✅ | — | — | ชื่อไฟล์ |
| `MIMETYPE` | — | ✅ | — | — | MIME type |
| `FILESIZE` | — | ✅ | — | — | ขนาดไฟล์ (bytes) |
| `DOCUMENT_ID` | — | ✅ | — | — | ID ของ document |
| `SOURCE_URL` | — | ✅ | — | — | URL ต้นทาง |

### Body Fields

| Field | Chat | Ingest | Nudges | URL | หมายเหตุ |
|-------|------|--------|--------|-----|---------|
| `input_value` | user message | `"Ingest files"` | `""` | URL string | ไหลเข้า ChatInput node |
| `input_type` | `"chat"` | `"chat"` | `"chat"` | `"chat"` | fixed |
| `output_type` | `"chat"` | `"text"` | `"chat"` | `"chat"` | fixed |
| `session_id` | user_id | upload_session | nudges_session | url_session | Langflow conversation ID |
| `stream` | true/false | false | false | false | SSE streaming |
| `tweaks` | optional | required* | — | — | *ต้องใส่ path ของไฟล์ |

### Tweaks Node IDs (สำหรับ Override)

| Node ID | Flow | Fields ที่ Override ได้ |
|---------|------|----------------------|
| `DoclingRemote-Dp3PX` | Ingest | `path` (list), `do_ocr`, `ocr_engine` |
| `SplitText-QIKhg` | Ingest, URL | `chunk_size`, `chunk_overlap`, `separator` |
| `Agent-Nfw7u` | Chat | `model_name`, `system_prompt`, `temperature` |
| `EmbeddingModel-*` | All | `model`, `api_key` |
| `OpenSearchVectorStore-*` | All | `filter_expression`, `number_of_results` |
| `LanguageModelComponent-*` | Nudges | `model_name`, `temperature` |

---

## ตัวอย่างการเรียก Flow แบบ Raw (เพื่อ debug / custom)

### Chat Flow โดยตรงจาก Langflow API

```bash
# ต้องมี Langflow API Key
curl -X POST http://localhost:7860/api/v1/run/1098eea1-6649-4e1d-aed1-b77249fb8dd0 \
  -H "Content-Type: application/json" \
  -H "x-api-key: <LANGFLOW_API_KEY>" \
  -H "x-langflow-global-var-jwt: <USER_JWT>" \
  -H "x-langflow-global-var-selected_embedding_model: text-embedding-3-small" \
  -H 'x-langflow-global-var-openrag-query-filter: {"filter": [], "limit": 10}' \
  -d '{
    "input_value": "นโยบายลาพักร้อนมีกี่วัน?",
    "input_type": "chat",
    "output_type": "chat",
    "session_id": "my_session_001"
  }'
```

### Ingest Flow โดยตรง

```bash
# Step 1: Upload ไฟล์ไปก่อน
FILE_ID=$(curl -s -X POST http://localhost:7860/api/v2/files \
  -H "x-api-key: <LANGFLOW_API_KEY>" \
  -F "file=@/path/to/document.pdf" | jq -r '.path')

# Step 2: Run ingestion flow พร้อม path + ACL
curl -X POST http://localhost:7860/api/v1/run/5488df7c-b93f-4f87-a446-b67028bc0813 \
  -H "Content-Type: application/json" \
  -H "x-api-key: <LANGFLOW_API_KEY>" \
  -H "x-langflow-global-var-jwt: <USER_JWT>" \
  -H "x-langflow-global-var-owner: user@company.com" \
  -H "x-langflow-global-var-owner_name: John Doe" \
  -H "x-langflow-global-var-owner_email: user@company.com" \
  -H 'x-langflow-global-var-allowed_users: ["ceo@company.com"]' \
  -H 'x-langflow-global-var-allowed_groups: ["hr", "all_employees"]' \
  -H "x-langflow-global-var-connector_type: upload" \
  -H "x-langflow-global-var-selected_embedding_model: text-embedding-3-small" \
  -d "{
    \"input_value\": \"Ingest files\",
    \"input_type\": \"chat\",
    \"output_type\": \"text\",
    \"session_id\": \"ingest_session_001\",
    \"tweaks\": {
      \"DoclingRemote-Dp3PX\": {
        \"path\": [\"$FILE_ID\"]
      }
    }
  }"
```

### ดู Flow Schema (รู้ inputs/outputs ทั้งหมดของ flow)

```bash
# ดู flow details + node schema
curl http://localhost:7860/api/v1/flows/1098eea1-6649-4e1d-aed1-b77249fb8dd0 \
  -H "x-api-key: <LANGFLOW_API_KEY>"

# ดู input schema เฉพาะ
curl http://localhost:7860/api/v1/flows/1098eea1-6649-4e1d-aed1-b77249fb8dd0/input_schema \
  -H "x-api-key: <LANGFLOW_API_KEY>"
```

---

## สรุป: ถ้าจะสร้าง Custom UI ให้ครบต้องทำอะไร

```
ขั้นต่ำที่ต้องทำ:
  1. ✅ Login → ได้ JWT (หรือใช้ no-auth mode)
  2. ✅ เรียกผ่าน Backend API เท่านั้น (อย่า bypass)
      → Backend จะ inject JWT + credentials ให้อัตโนมัติ
  3. ✅ ส่ง Cookie: credentials:'include' ทุก request

สิ่งที่ Backend จัดการให้อัตโนมัติ:
  ✅ Inject JWT header ไปยัง Langflow
  ✅ Inject embedding model + provider credentials
  ✅ Build filter expression จาก user request
  ✅ OpenSearch ACL filtering ตาม user identity

สิ่งที่ต้อง handle เองใน Custom UI:
  ✅ Login flow (OAuth redirect)
  ✅ Handle SSE stream สำหรับ streaming response
  ✅ Display sources จาก response
  ✅ Upload form สำหรับ document ingestion
  ❌ ไม่ต้อง handle JWT validation
  ❌ ไม่ต้อง handle permission filtering
```

---

*อ้างอิง source code:*
- `src/services/langflow_file_service.py` (line 138–164) — headers สำหรับ ingest
- `src/services/chat_service.py` (line 69–134) — headers สำหรับ chat
- `flows/openrag_agent.json` — Chat flow node structure
- `flows/ingestion_flow.json` — Ingestion flow node structure
- `flows/openrag_nudges.json` — Nudges flow node structure
- `flows/openrag_url_mcp.json` — URL ingestion flow node structure
