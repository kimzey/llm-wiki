# Phase 5: Integration — ทุก Service ทำงานร่วมกัน + ตัวอย่างจริง

## ภาพรวม Integration ทั้งหมด

```
                        ┌─────────────────┐
                        │   User/Browser  │
                        └────────┬────────┘
                                 │ HTTP
                        ┌────────▼────────┐
                        │    Frontend     │
                        │   (Next.js)     │
                        │   Port: 3000    │
                        └────────┬────────┘
                                 │ HTTP/REST
                        ┌────────▼────────┐
                        │    Backend      │◄──── Public API (v1)
                        │   (FastAPI)     │      API Key Auth
                        │   Port: 8080    │
                        └───┬─────────┬───┘
                            │         │
              ┌─────────────┘         └──────────────┐
              │ HTTP                             HTTP │
    ┌─────────▼─────────┐              ┌─────────────▼─────┐
    │      Langflow     │              │    OpenSearch      │
    │  Port: 7860       │◄────────────►│    Port: 9200      │
    │  (Workflows)      │    Direct    │  (Vector DB)       │
    └────────┬──────────┘   Indexing  └────────────────────┘
             │ HTTP
    ┌────────▼──────────┐
    │      Docling      │
    │  Port: 5001       │
    │  (Doc Parser)     │
    └───────────────────┘
```

---

## Scenario 1: อัปโหลดและ Index เอกสาร HR Policy

### Context:
- HR Manager อัปโหลด "hr_policy_2024.pdf" (30 หน้า)
- ต้องการให้พนักงานทั้งหมดในบริษัทค้นหาได้

### Step-by-Step:

**Step 1: Upload via Frontend**
```
User Action: เลือกไฟล์ → คลิก Upload
Browser: POST http://localhost:3000/api/upload
  Body: FormData { file: hr_policy_2024.pdf }
```

**Step 2: Backend รับและสร้าง Task**
```python
# POST /upload (Backend)
task_id = "task-hr-001"
task = {
    "task_id": task_id,
    "status": "pending",
    "filename": "hr_policy_2024.pdf",
    "owner": "hr.manager@company.com"
}
# บันทึกไฟล์ temp: /tmp/openrag/hr_policy_2024.pdf
```

**Step 3: Upload ไปที่ Langflow**
```python
# Backend → Langflow Files API
POST http://langflow:7860/api/v1/files/upload/{ingest_flow_id}
Headers: Authorization: Bearer {langflow_api_key}
Body: FormData { file: hr_policy_2024.pdf }

Response: {
  "file_path": "/var/langflow/uploads/hr_policy_2024.pdf"
}
```

**Step 4: Trigger Ingestion Flow**
```python
# Backend → Langflow Run
POST http://langflow:7860/api/v1/run/{ingest_flow_id}
Body: {
  "input_value": "/var/langflow/uploads/hr_policy_2024.pdf",
  "tweaks": {
    "DoclingComponent-abc1": {
      "docling_serve_url": "http://host.docker.internal:5001"
    },
    "OpenSearchVectorStore-def2": {
      "index_name": "documents",
      "username": "admin",
      "password": "YourPassword"
    },
    "MetadataInjector-ghi3": {
      "owner": "hr.manager@company.com",
      "allowed_users": [],
      "allowed_groups": ["all-employees"],
      "filename": "hr_policy_2024.pdf",
      "document_id": "hr-policy-2024"
    }
  }
}
```

**Step 5: Langflow ทำงาน**
```
[Langflow Ingestion Flow]

Node 1: File Input
  → ไฟล์: hr_policy_2024.pdf (2.1 MB, 30 หน้า)

Node 2: Docling Component
  → POST http://host.docker.internal:5001/convert/source
  → Processing: Layout analysis, table detection
  → Time: 8.3 seconds
  → Output: 15,420 words Markdown

Node 3: RecursiveCharacterTextSplitter
  → chunk_size: 1000, overlap: 200
  → Output: 62 chunks

Node 4: OpenAIEmbeddings
  → Model: text-embedding-3-small
  → API calls: 62 (batch)
  → Output: 62 × [1536 floats]

Node 5: OpenSearchVectorStore
  → Bulk index: 62 documents
  → Index: documents
  → Time: 1.2 seconds
```

**Step 6: OpenSearch Index ผลลัพธ์**
```json
// ตัวอย่าง 2 chunks ที่ถูก index:

{
  "_id": "hr-policy-2024-chunk-001",
  "_source": {
    "document_id": "hr-policy-2024",
    "filename": "hr_policy_2024.pdf",
    "page": 3,
    "text": "# Leave Policies\n\n## Sick Leave\nEmployees are entitled to 30 days of sick leave per year. For absences exceeding 3 consecutive days, a medical certificate is required...",
    "embeddings": [0.0234, -0.1892, 0.4521, 0.0012, ...],
    "owner": "hr.manager@company.com",
    "allowed_users": [],
    "allowed_groups": ["all-employees"],
    "created_time": "2024-11-15T10:30:00Z"
  }
}

{
  "_id": "hr-policy-2024-chunk-002",
  "_source": {
    "document_id": "hr-policy-2024",
    "filename": "hr_policy_2024.pdf",
    "page": 5,
    "text": "## Annual Leave\nAll permanent employees receive 15 days of annual leave. Employees with more than 5 years of service receive 20 days...",
    "embeddings": [-0.0891, 0.3241, -0.1234, 0.0567, ...],
    ...
  }
}
```

**Step 7: Task Complete**
```json
// GET /tasks/task-hr-001
{
  "task_id": "task-hr-001",
  "status": "success",
  "result": {
    "filename": "hr_policy_2024.pdf",
    "chunks_indexed": 62,
    "processing_time": 45.2
  }
}
```

---

## Scenario 2: พนักงานถามคำถามเกี่ยวกับ HR Policy

### Context:
- พนักงาน "สมชาย" อยากรู้เรื่องลาป่วย

**Step 1: User ส่งคำถาม**
```
User: "ถ้าป่วย 5 วันต้องทำอย่างไรบ้าง?"
POST /langflow/chat
Body: {
  "message": "ถ้าป่วย 5 วันต้องทำอย่างไรบ้าง?",
  "session_id": "sess-somchai-001",
  "user": "somchai@company.com"
}
```

**Step 2: Backend ส่งไป Langflow**
```python
# POST http://langflow:7860/api/v1/run/{chat_flow_id}?stream=true
{
  "input_value": "ถ้าป่วย 5 วันต้องทำอย่างไรบ้าง?",
  "input_type": "chat",
  "session_id": "sess-somchai-001",
  "tweaks": {
    "OpenSearchRetriever-xxx": {
      "filter_user": "somchai@company.com",
      "filter_groups": ["all-employees"],
      "k": 5
    }
  }
}
```

**Step 3: Langflow Agent ค้นหา**
```
[Chat Flow Node 1: Embedding]
  query = "ถ้าป่วย 5 วันต้องทำอย่างไรบ้าง?"
  query_vector = OpenAI.embed(query)
  → [0.0123, -0.2341, 0.5678, ...]

[Chat Flow Node 2: OpenSearch KNN Search]
  POST https://opensearch:9200/documents/_search
  {
    "size": 5,
    "query": {
      "bool": {
        "must": {
          "knn": {
            "embeddings": {
              "vector": [0.0123, -0.2341, 0.5678, ...],
              "k": 5
            }
          }
        },
        "filter": {
          "bool": {
            "should": [
              {"term": {"allowed_users": "somchai@company.com"}},
              {"terms": {"allowed_groups": ["all-employees"]}}
            ]
          }
        }
      }
    }
  }
```

**Step 4: OpenSearch ส่งผลลัพธ์**
```json
{
  "hits": {
    "hits": [
      {
        "_score": 0.943,
        "_source": {
          "text": "## Sick Leave\nEmployees are entitled to 30 days of sick leave per year. For absences exceeding 3 consecutive days, a medical certificate is required from a licensed physician.",
          "filename": "hr_policy_2024.pdf",
          "page": 3
        }
      },
      {
        "_score": 0.891,
        "_source": {
          "text": "### Medical Certificate Requirements\nWhen an employee is absent due to illness for more than 3 consecutive days, they must submit a medical certificate within 7 days of returning to work.",
          "filename": "hr_policy_2024.pdf",
          "page": 4
        }
      },
      {
        "_score": 0.834,
        "_source": {
          "text": "### Sick Leave Documentation\nEmployees should notify their supervisor on the first day of absence. Extended sick leave requires HR approval.",
          "filename": "hr_policy_2024.pdf",
          "page": 4
        }
      }
    ]
  }
}
```

**Step 5: LLM Generate Response**
```
[Chat Flow Node 3: LLM Prompt]

System: "You are a helpful HR assistant. Answer questions based on the provided context only."

Context:
  [Doc 1 - hr_policy_2024.pdf, p.3, score: 0.943]
  "Employees are entitled to 30 days of sick leave per year..."

  [Doc 2 - hr_policy_2024.pdf, p.4, score: 0.891]
  "When an employee is absent due to illness for more than 3 consecutive days..."

  [Doc 3 - hr_policy_2024.pdf, p.4, score: 0.834]
  "Employees should notify their supervisor on the first day..."

User: "ถ้าป่วย 5 วันต้องทำอย่างไรบ้าง?"

[LLM Response - Streaming]:
"ถ้าคุณป่วย 5 วัน มีสิ่งที่ต้องทำดังนี้:

1. **แจ้งหัวหน้างาน** ในวันแรกที่ลาป่วย

2. **ใบรับรองแพทย์** — เนื่องจากลาเกิน 3 วัน ต้องมีใบรับรองแพทย์
   จากแพทย์ที่มีใบอนุญาต และต้องส่งภายใน 7 วัน
   หลังจากกลับมาทำงาน

3. **สิทธิลาป่วย** — บริษัทให้สิทธิลาป่วย 30 วันต่อปี
   การลา 5 วันยังอยู่ในขอบเขตสิทธิที่ได้รับ

📄 *อ้างอิง: HR Policy 2024 หน้า 3-4*"
```

---

## Scenario 3: ใช้ Public API เพื่อ Integrate กับ App อื่น

### Context:
- บริษัทต้องการ integrate OpenRAG เข้ากับ Internal Chatbot ที่มีอยู่แล้ว

**Step 1: สร้าง API Key**
```
Dashboard → Settings → API Keys → Create New Key
→ Key: "sk-openrag-xxxxxxxxxxxxxxxx"
```

**Step 2: เรียกใช้ Chat API**
```bash
curl -X POST "http://your-openrag.internal:8080/v1/chat" \
  -H "Authorization: Bearer sk-openrag-xxxxxxxxxxxxxxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "What is our refund policy?",
    "session_id": "external-bot-session-123",
    "knowledge_filter_id": "customer-facing-docs"
  }'
```

**Response:**
```json
{
  "chat_id": "chat-abc-123",
  "response": "Our refund policy allows returns within 30 days of purchase...",
  "citations": [
    {
      "filename": "refund_policy.pdf",
      "page": 2,
      "text": "Returns are accepted within 30 days..."
    }
  ],
  "session_id": "external-bot-session-123"
}
```

**Step 3: Ingest เอกสารผ่าน API**
```bash
curl -X POST "http://your-openrag.internal:8080/v1/documents/ingest" \
  -H "Authorization: Bearer sk-openrag-xxxxxxxxxxxxxxxx" \
  -F "file=@new_policy_2025.pdf" \
  -F "allowed_groups=customer-service"
```

---

## Scenario 4: Knowledge Filters — จำกัด Scope การค้นหา

### Context:
- ต้องการให้ HR Bot ค้นหาเฉพาะเอกสาร HR
- ไม่ต้องการให้ปะปนกับเอกสาร Finance

**Step 1: สร้าง Knowledge Filter**
```bash
POST /knowledge-filter
{
  "name": "HR Documents",
  "description": "Only HR-related documents",
  "filter": {
    "terms": {
      "allowed_groups": ["hr-team", "all-employees"]
    }
  }
}
# Response: { "filter_id": "kf-hr-001" }
```

**Step 2: ใช้ Filter ใน Chat**
```bash
POST /langflow/chat
{
  "message": "ลาป่วยได้กี่วัน?",
  "knowledge_filter_id": "kf-hr-001"
}
# → จะค้นหาเฉพาะเอกสาร HR เท่านั้น
```

---

## Scenario 5: Google Drive Connector

### Context:
- ต้องการ sync เอกสารจาก Google Drive อัตโนมัติ

**Setup:**
```bash
# .env
GOOGLE_OAUTH_CLIENT_ID=your-client-id
GOOGLE_OAUTH_CLIENT_SECRET=your-client-secret
```

**Flow:**
```
1. User เชื่อมต่อ Google Drive ผ่าน OAuth
   GET /connectors/google_drive/auth

2. เลือก Folder ที่ต้องการ sync

3. Trigger sync:
   POST /connectors/google_drive/sync
   { "folder_id": "1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs" }

4. ระบบ:
   - Download ไฟล์ทั้งหมดใน folder
   - ส่งแต่ละไฟล์ผ่าน Ingestion Pipeline
   - Index ลง OpenSearch

5. ตรวจสอบ status:
   GET /connectors/google_drive/status
   → { "status": "synced", "files_count": 47, "last_sync": "2024-11-15T..." }
```

---

## Environment Variables — ครบทุก Service

```bash
# ============= CORE =============
SECRET_KEY=your-super-secret-key-min-32-chars
NEXT_PUBLIC_API_URL=http://localhost:8080

# ============= OPENSEARCH =============
OPENSEARCH_HOST=opensearch
OPENSEARCH_PORT=9200
OPENSEARCH_USERNAME=admin
OPENSEARCH_PASSWORD=YourStrongP@ssw0rd!
OPENSEARCH_INDEX_NAME=documents
OPENSEARCH_DATA_PATH=./opensearch-data
OPENSEARCH_INITIAL_ADMIN_PASSWORD=YourStrongP@ssw0rd!

# ============= LANGFLOW =============
LANGFLOW_URL=http://langflow:7860
LANGFLOW_SECRET_KEY=langflow-secret
LANGFLOW_AUTO_LOGIN=False
LANGFLOW_TIMEOUT=2400
LANGFLOW_CONNECT_TIMEOUT=30
LANGFLOW_INGEST_FLOW_ID=5488df7c-b93f-4f87-a446-b67028bc0813
LANGFLOW_CHAT_FLOW_ID=1098eea1-6649-4e1d-aed1-b77249fb8dd0
LANGFLOW_URL_INGEST_FLOW_ID=72c3d17c-2dac-4a73-b48a-6518473d7830
NUDGES_FLOW_ID=ebc01d31-1976-46ce-a385-b0240327226c

# ============= DOCLING =============
DOCLING_SERVE_URL=http://host.docker.internal:5001
DOCLING_OCR_ENGINE=easyocr

# ============= LLM PROVIDERS (เลือกอย่างน้อย 1) =============
# Option A: OpenAI (แนะนำสำหรับเริ่มต้น)
OPENAI_API_KEY=sk-proj-...

# Option B: Anthropic Claude
ANTHROPIC_API_KEY=sk-ant-...

# Option C: IBM WatsonX
WATSONX_API_KEY=...
WATSONX_ENDPOINT=https://us-south.ml.cloud.ibm.com
WATSONX_PROJECT_ID=...

# Option D: Ollama (Local LLM ไม่มีค่าใช้จ่าย)
OLLAMA_ENDPOINT=http://host.docker.internal:11434

# ============= EMBEDDING =============
SELECTED_EMBEDDING_MODEL=text-embedding-3-small

# ============= INGESTION =============
DISABLE_INGEST_WITH_LANGFLOW=false
UPLOAD_BATCH_SIZE=25
INGESTION_TIMEOUT=3600
INGEST_SAMPLE_DATA=true

# ============= AUTH (Optional OAuth) =============
GOOGLE_OAUTH_CLIENT_ID=
GOOGLE_OAUTH_CLIENT_SECRET=
MICROSOFT_GRAPH_OAUTH_CLIENT_ID=
MICROSOFT_GRAPH_OAUTH_CLIENT_SECRET=
```

---

## API Endpoints ครบทุกอัน

### Chat
```
POST   /chat                    # Chat แบบ traditional
POST   /langflow                # Chat ผ่าน Langflow (แนะนำ)
GET    /chat/history            # ดูประวัติ chat
GET    /langflow/history        # ดูประวัติ Langflow chat
DELETE /sessions/{session_id}   # ลบ session
```

### Documents
```
POST   /upload                           # อัปโหลดไฟล์
POST   /upload_ingest_router             # อัปโหลด + ingest (smart routing)
GET    /documents/check-filename         # ตรวจว่าไฟล์มีอยู่แล้วไหม
POST   /documents/delete-by-filename     # ลบเอกสาร
```

### Langflow
```
POST   /langflow/files/upload    # Upload file ไป Langflow
POST   /langflow/ingest          # Run ingestion flow
POST   /langflow/upload_ingest   # Upload + Ingest รวม
DELETE /langflow/files           # ลบไฟล์ใน Langflow
```

### Tasks
```
GET    /tasks                    # ดู task ทั้งหมด
GET    /tasks/{task_id}          # ดู task นึง
POST   /tasks/{task_id}/cancel   # ยกเลิก task
```

### Search
```
POST   /search    # Semantic search
```

### Knowledge Filters
```
POST   /knowledge-filter               # สร้าง filter
POST   /knowledge-filter/search        # ค้นหา filter
GET    /knowledge-filter/{id}          # ดู filter
PUT    /knowledge-filter/{id}          # แก้ไข filter
DELETE /knowledge-filter/{id}          # ลบ filter
```

### Connectors
```
GET    /connectors                     # ดู connectors ที่มี
POST   /connectors/{type}/sync         # sync connector
GET    /connectors/{type}/status       # ดู status
DELETE /connectors/{type}/disconnect   # disconnect
```

### Auth
```
POST   /auth/init        # เริ่ม OAuth flow
POST   /auth/callback    # OAuth callback
GET    /auth/me          # ดู current user
POST   /auth/logout      # logout
```

### Flows
```
POST   /reset-flow/{flow_type}    # reset flow (nudges/retrieval/ingest)
```

### Models & Settings
```
GET    /models              # ดู available models
GET    /settings            # ดู settings
PUT    /settings            # แก้ไข settings
```

### Health
```
GET    /health              # ดู service health
GET    /docling/health      # ดู Docling health
```

### Public API v1
```
POST   /v1/chat                 # Chat (API key auth)
GET    /v1/chat/{chat_id}       # ดู chat
DELETE /v1/chat/{chat_id}       # ลบ chat
POST   /v1/search               # Search (API key auth)
POST   /v1/documents/ingest     # Ingest ผ่าน API
GET    /v1/tasks/{task_id}      # ดู task status
DELETE /v1/documents            # ลบ documents
```

---

## Troubleshooting — ปัญหาที่พบบ่อย

### 1. Docling ไม่ตอบสนอง
```bash
# ตรวจสอบ Docling health
curl http://localhost:5001/health

# ถ้าไม่ตอบ — รัน Docling
docker run -p 5001:5001 quay.io/docling-project/docling-serve:latest

# ใน .env ตรวจสอบ:
DOCLING_SERVE_URL=http://host.docker.internal:5001
```

### 2. OpenSearch ไม่ start
```bash
# ตรวจสอบ password — ต้องมีความซับซ้อน
OPENSEARCH_PASSWORD=At_Least_8_Chars_With_Upper_Lower_Number!

# ตรวจสอบ memory
sudo sysctl -w vm.max_map_count=262144
```

### 3. Langflow Flow ไม่รัน
```bash
# ตรวจสอบ Flow IDs ใน .env
LANGFLOW_INGEST_FLOW_ID=...

# Reset flow ผ่าน API
POST /reset-flow/ingest
```

### 4. Embedding Dimension Mismatch
```
Error: Vector dimension mismatch
Cause: เปลี่ยน embedding model โดยไม่ re-index

Fix: ลบ index เก่า + re-index ทั้งหมด
DELETE https://localhost:9200/documents
# แล้ว ingest เอกสารใหม่ทั้งหมด
```

### 5. Ingestion Timeout
```bash
# เพิ่ม timeout สำหรับไฟล์ขนาดใหญ่
LANGFLOW_TIMEOUT=7200    # 2 ชั่วโมง
INGESTION_TIMEOUT=7200
```

---

## Performance Tips

### สำหรับเอกสารจำนวนมาก:
```bash
# อัปโหลดเป็น batch
UPLOAD_BATCH_SIZE=25  # ทำงานพร้อมกัน 25 ไฟล์

# ใช้ GPU กับ Docling
docker run --gpus all -p 5001:5001 \
  quay.io/docling-project/docling-serve:latest-gpu
```

### สำหรับ Search ที่เร็วขึ้น:
```json
// เพิ่ม ef_search ใน OpenSearch index
PUT /documents/_settings
{
  "index.knn.algo_param.ef_search": 200
}
```

### สำหรับ LLM ที่ถูกลง:
```bash
# ใช้ GPT-4o-mini แทน GPT-4o
# แก้ใน Langflow flow หรือ config.yaml

# หรือใช้ Ollama (ฟรี)
OLLAMA_ENDPOINT=http://host.docker.internal:11434
# แล้วเลือก model ใน Langflow flow
```

---

## สรุป: ความสัมพันธ์ของทุก Service

```
Service       | ทำหน้าที่           | Input            | Output
─────────────────────────────────────────────────────────────────
Docling       | Parse เอกสาร       | PDF/DOCX/ภาพ    | Markdown/JSON
Langflow      | Orchestrate flow   | File/Query       | Indexed/Response
OpenSearch    | Store & Search     | Vectors          | Ranked chunks
Backend       | API Gateway        | HTTP requests    | JSON responses
Frontend      | User Interface     | User actions     | UI updates
```

**Docling** ทำให้เอกสารอ่านได้ →
**Langflow** จัดการ pipeline →
**OpenSearch** เก็บและค้นหา →
**Backend** ควบคุมทั้งหมด →
**Frontend** แสดงผลให้ User

---

*กลับไป: [Phase 4 — Langflow](./phase4-langflow.md)*
*เริ่มต้น: [Phase 1 — Overview](./phase1-overview.md)*
