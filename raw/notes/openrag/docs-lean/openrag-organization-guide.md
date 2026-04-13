# OpenRAG — คู่มือการใช้งานในองค์กร (ฉบับละเอียด)

> เอกสารนี้ตอบคำถามหลัก 4 ข้อ:
> 1. ต้องทำอะไรเพิ่มเพื่อให้ระบบ RAG ขององค์กรใช้งานได้?
> 2. Start service แล้วใช้ได้เลยไหม หรือต้องตั้งค่าเพิ่ม?
> 3. ระดับการเข้าถึงข้อมูลตาม Role ทำงานอย่างไร?
> 4. ต้องการคนดูแลต่อเนื่องไหม หรือทำครั้งเดียวจบ?

---

## สารบัญ

- [Architecture ภาพรวม](#architecture-ภาพรวม)
- [1. Start Service แล้วต้องทำอะไรเพิ่ม?](#1-start-service-แล้วต้องทำอะไรเพิ่ม)
- [2. การนำข้อมูลเข้า (Data Ingestion)](#2-การนำข้อมูลเข้า-data-ingestion)
- [3. ระดับการเข้าถึงข้อมูลตาม Role](#3-ระดับการเข้าถึงข้อมูลตาม-role)
- [4. ต้องการคนดูแลต่อเนื่องไหม?](#4-ต้องการคนดูแลต่อเนื่องไหม)
- [ตัวอย่างข้อมูลจริง](#ตัวอย่างข้อมูลจริง)
- [Checklist ก่อน Go Live](#checklist-ก่อน-go-live)

---

## Architecture ภาพรวม

```
┌─────────────────────────────────────────────────────────────┐
│                        OpenRAG Stack                        │
├──────────────┬──────────────┬──────────────┬───────────────┤
│  Frontend    │   Backend    │   Langflow   │  OpenSearch   │
│  Next.js     │   FastAPI    │  (Workflow)  │  (Vector DB)  │
│  Port: 3000  │  Port: 8000  │  Port: 7860  │  Port: 9200   │
└──────────────┴──────────────┴──────────────┴───────────────┘
                        ↕ Docling (Parser)
                        Port: 5001 (PDF/DOCX → Markdown)
```

**Services หลัก 5 ตัว ที่ต้องรันพร้อมกัน:**

| Service | หน้าที่ | Image |
|---------|---------|-------|
| `backend` | API + Auth + RAG Logic | `langflowai/openrag-backend` |
| `frontend` | Web UI | `langflowai/openrag-frontend` |
| `langflow` | Workflow orchestration | `langflowai/openrag-langflow` |
| `opensearch` | Vector Database | `langflowai/openrag-opensearch` |
| `docling` | Document Parser | (built-in หรือ separate) |

---

## 1. Start Service แล้วต้องทำอะไรเพิ่ม?

### คำตอบสั้น: **ต้องทำอีก 3 ขั้นตอนหลัก** ก่อนใช้งานจริงได้

```
docker compose up -d
       ↓
  [ยังใช้ไม่ได้]
       ↓
  Step 1: ตั้งค่า LLM Provider (API Key)
       ↓
  Step 2: ตั้งค่า Embedding Model
       ↓
  Step 3: นำข้อมูลองค์กรเข้า
       ↓
  [ใช้งานได้แล้ว]
```

---

### Step 1: ตั้งค่า Environment Variables

สร้างไฟล์ `.env` ในโฟลเดอร์ root:

```env
# ===== LLM Provider (เลือกอย่างน้อย 1 อัน) =====
OPENAI_API_KEY=sk-proj-xxxxxxxxxxxxxxxxxxxxxxxx

# หรือ Anthropic
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxxxxxxxxxxxxx

# หรือ IBM WatsonX (on-premise friendly)
WATSONX_API_KEY=xxxxxxxxxxxxxxxx
WATSONX_ENDPOINT=https://api.dataplatform.cloud.ibm.com
WATSONX_PROJECT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# หรือ Ollama (local, ไม่ต้องจ่าย)
OLLAMA_ENDPOINT=http://host.docker.internal:11434

# ===== OpenSearch (เปลี่ยน password จาก default!) =====
OPENSEARCH_HOST=opensearch
OPENSEARCH_PORT=9200
OPENSEARCH_USERNAME=admin
OPENSEARCH_PASSWORD=MyStr0ng!Pass@2024   # <<< เปลี่ยนตรงนี้

# ===== Langflow =====
LANGFLOW_URL=http://langflow:7860
LANGFLOW_SUPERUSER=admin
LANGFLOW_SUPERUSER_PASSWORD=MyLangflow!Pass@2024  # <<< เปลี่ยนตรงนี้

# ===== OAuth (ถ้าต้องการ Google/Microsoft Login) =====
GOOGLE_OAUTH_CLIENT_ID=xxxxxxxxxx.apps.googleusercontent.com
GOOGLE_OAUTH_CLIENT_SECRET=GOCSPX-xxxxxxxxxxxxxxxxxx

MICROSOFT_GRAPH_OAUTH_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
MICROSOFT_GRAPH_OAUTH_CLIENT_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxx

# ===== ถ้าไม่ต้องการ OAuth (no-auth mode) =====
# ไม่ต้องใส่ GOOGLE_OAUTH_CLIENT_ID → ระบบเป็น anonymous mode อัตโนมัติ
```

---

### Step 2: Onboarding ผ่าน UI

หลัง docker compose up เข้า `http://localhost:3000` → จะมี Onboarding Wizard:

```
Step 1: เลือก LLM Provider
        ├── OpenAI → ใส่ API Key → เลือก model (gpt-4o, gpt-4, etc.)
        ├── Anthropic → ใส่ API Key
        ├── WatsonX → ใส่ credentials
        └── Ollama → ใส่ endpoint

Step 2: เลือก Embedding Model
        ├── text-embedding-3-small (OpenAI) ← แนะนำ
        ├── text-embedding-3-large (OpenAI) ← แม่นกว่า แต่แพงกว่า
        └── nomic-embed-text (Ollama) ← ฟรี local

Step 3: Test Connection
Step 4: เริ่มใช้งาน
```

หรือตั้งค่าผ่าน API:

```bash
curl -X POST http://localhost:8000/onboarding \
  -H "Content-Type: application/json" \
  -d '{
    "provider": "openai",
    "api_key": "sk-proj-xxx",
    "embedding_model": "text-embedding-3-small",
    "llm_model": "gpt-4o"
  }'
```

---

### Step 3: config.yaml ที่ได้หลัง Onboarding

ระบบจะสร้างไฟล์ `./config/config.yaml` อัตโนมัติ:

```yaml
providers:
  openai:
    api_key: "sk-proj-xxxxxxxx"
    configured: true
  anthropic:
    api_key: ""
    configured: false

knowledge:
  embedding_model: "text-embedding-3-small"
  embedding_provider: "openai"
  chunk_size: 1000          # ขนาด chunk ของแต่ละ piece
  chunk_overlap: 200        # overlap ระหว่าง chunk
  table_structure: true     # Parse ตาราง PDF
  ocr: false                # OCR สำหรับรูปภาพ
  picture_descriptions: false

agent:
  llm_model: "gpt-4o"
  llm_provider: "openai"
  system_prompt: "You are a helpful assistant..."

onboarding:
  current_step: 4
  edited: true
```

---

## 2. การนำข้อมูลเข้า (Data Ingestion)

### รูปแบบไฟล์ที่รองรับ

| ประเภท | Extension | หมายเหตุ |
|--------|-----------|----------|
| PDF | `.pdf` | รองรับ table extraction, OCR option |
| Word | `.docx` | รองรับ header, table, image |
| PowerPoint | `.pptx` | แต่ละ slide เป็น chunk แยก |
| รูปภาพ | `.png`, `.jpg`, `.jpeg` | ต้องเปิด OCR |
| Web URL | `https://...` | ผ่าน URL ingestion flow |
| Google Drive | (connector) | ต้องตั้งค่า OAuth |
| OneDrive | (connector) | ต้องตั้งค่า OAuth |
| SharePoint | (connector) | ต้องตั้งค่า OAuth |
| S3 Bucket | (connector) | ต้องตั้งค่า AWS credentials |

---

### วิธีที่ 1: Upload ผ่าน Web UI

```
1. เข้า http://localhost:3000
2. คลิก "Upload Documents"
3. Drag & Drop ไฟล์ หรือ Browse
4. เลือก Knowledge Filter (ถ้ามี)
5. คลิก "Upload & Ingest"
6. รอ Processing (ดู progress ได้ที่ Tasks)
```

---

### วิธีที่ 2: Upload ผ่าน API (สำหรับ Automation)

```bash
# Upload ไฟล์เดี่ยว
curl -X POST http://localhost:8000/router/upload_ingest \
  -H "Authorization: Bearer <jwt_token>" \
  -F "file=@/path/to/document.pdf" \
  -F "knowledge_filter_id=kf_engineering_docs"

# Response
{
  "task_id": "task_abc123",
  "status": "processing",
  "filename": "document.pdf"
}

# ตรวจสอบ status
curl http://localhost:8000/tasks/task_abc123 \
  -H "Authorization: Bearer <jwt_token>"

# Response
{
  "task_id": "task_abc123",
  "status": "completed",
  "chunks_indexed": 47,
  "processing_time": 12.3
}
```

---

### วิธีที่ 3: Public API v1 (สำหรับ Integration)

```bash
# สร้าง API Key ก่อน
curl -X POST http://localhost:8000/keys \
  -H "Authorization: Bearer <jwt_token>" \
  -d '{"name": "My Integration Key"}'

# Response
{
  "key_id": "key_xyz789",
  "api_key": "orag_abc123...xxxxxxxx",  # เก็บไว้ จะเห็นครั้งเดียว!
  "prefix": "orag_abc1"
}

# ใช้ API Key สำหรับ Ingest
curl -X POST http://localhost:8000/v1/documents/ingest \
  -H "X-API-Key: orag_abc123...xxxxxxxx" \
  -F "file=@annual_report_2024.pdf"
```

---

### Pipeline การ Ingest (สิ่งที่เกิดขึ้นเบื้องหลัง)

```
ไฟล์ PDF "นโยบาย_HR.pdf"
         │
         ▼
[Langflow รับไฟล์ชั่วคราว]
         │
         ▼
[Docling แปลง PDF → Markdown]
  "## นโยบายการลา\n\nพนักงานมีสิทธิ์ลาพักร้อน..."
         │
         ▼
[Text Splitter แบ่งเป็น Chunks]
  Chunk 1: "นโยบายการลา พนักงานมีสิทธิ์ลาพักร้อน 10 วัน..."
  Chunk 2: "การลาป่วย พนักงานสามารถลาป่วยได้ไม่เกิน..."
  Chunk 3: ...
         │
         ▼
[Embedding Generation]
  Chunk 1 → [0.023, -0.156, 0.891, ...] (1536 dimensions)
  Chunk 2 → [0.445, 0.023, -0.234, ...]
         │
         ▼
[เก็บใน OpenSearch]
  {
    "document_id": "doc_abc123",
    "filename": "นโยบาย_HR.pdf",
    "text": "นโยบายการลา พนักงานมีสิทธิ์...",
    "chunk_embedding": [0.023, -0.156, 0.891, ...],
    "owner": "hr@company.com",
    "allowed_groups": ["hr", "all_employees"],
    "page": 1
  }
         │
         ▼
[ลบไฟล์ชั่วคราวจาก Langflow]
         │
         ▼
[พร้อมค้นหา!]
```

---

### ตัวอย่าง Chunk ที่เก็บใน OpenSearch

```json
{
  "_index": "documents",
  "_id": "chunk_7f3a9b2c",
  "_source": {
    "document_id": "doc_นโยบาย_HR_2024",
    "filename": "นโยบาย_HR_2024.pdf",
    "mimetype": "application/pdf",
    "page": 3,
    "text": "พนักงานมีสิทธิ์ลาพักร้อนประจำปีไม่เกิน 10 วันทำการ โดยต้องแจ้งล่วงหน้าอย่างน้อย 3 วัน และได้รับการอนุมัติจากหัวหน้างาน",
    "chunk_embedding_text-embedding-3-small": [0.023, -0.156, 0.891, 0.445, ...],
    "embedding_model": "text-embedding-3-small",
    "connector_type": "upload",
    "owner": "hr@company.com",
    "allowed_users": [],
    "allowed_groups": ["hr", "all_employees"],
    "user_permissions": {},
    "group_permissions": {},
    "created_time": "2024-01-15T09:30:00Z",
    "indexed_time": "2024-01-15T09:30:12Z",
    "metadata": {
      "source": "นโยบาย_HR_2024.pdf",
      "chunk_index": 12
    }
  }
}
```

---

## 3. ระดับการเข้าถึงข้อมูลตาม Role

### ระบบ Access Control ของ OpenRAG

OpenRAG ใช้ **Document-Level Access Control** — ทุก chunk มีข้อมูลว่าใครเข้าถึงได้:

```
┌─────────────────────────────────────────────────────┐
│                   Document Chunk                    │
├─────────────────────────────────────────────────────┤
│  owner: "hr@company.com"                            │
│  allowed_users: ["ceo@company.com"]                 │
│  allowed_groups: ["hr", "all_employees"]            │
└─────────────────────────────────────────────────────┘

ใครเข้าถึงได้?
  ✓ hr@company.com (owner)
  ✓ ceo@company.com (allowed_users)
  ✓ ทุกคนที่อยู่ใน group "hr"
  ✓ ทุกคนที่อยู่ใน group "all_employees"
  ✗ คนอื่นที่ไม่อยู่ใน list → ไม่เห็นเอกสารนี้เลย
```

---

### ตารางระดับ Role ในองค์กร (ตัวอย่าง)

| Role | เข้าถึงเอกสารอะไร | ทำอะไรได้ |
|------|-------------------|-----------|
| **HR Admin** | เอกสาร HR ทั้งหมด + นโยบาย | Upload, Share, Delete เอกสาร HR |
| **IT Admin** | เอกสาร IT + Infrastructure | Upload, Share เอกสาร IT |
| **Finance** | รายงานการเงิน + Budget | Upload เอกสาร Finance |
| **Engineer** | Technical Docs + Code Standards | Upload เอกสาร Engineering |
| **All Employees** | นโยบายทั่วไป + FAQ | ค้นหาได้ ไม่สามารถ Upload |
| **Manager** | เอกสารของทีมตัวเอง | ค้นหา + ดู analytics |
| **Guest/Temp** | เฉพาะที่ได้รับสิทธิ์ชัดเจน | ค้นหาเท่านั้น |

---

### วิธีตั้งค่า Access Control เมื่อ Upload

#### ผ่าน Web UI:
```
Upload Document → ตั้งค่า Sharing:
  □ Public (ทุกคนใน org)
  □ Private (เฉพาะฉัน)
  □ Specific Users → hr@company.com, ceo@company.com
  □ Groups → engineering, finance
```

#### ผ่าน API:
```bash
curl -X POST http://localhost:8000/router/upload_ingest \
  -H "Authorization: Bearer <token>" \
  -F "file=@confidential_report.pdf" \
  -F 'metadata={
    "allowed_users": ["ceo@company.com", "cfo@company.com"],
    "allowed_groups": ["executive", "finance"]
  }'
```

---

### Knowledge Filters — การกำหนด Context ให้ชัดเจน

Knowledge Filter คือ "กล่อง" ที่รวมเอกสารที่เกี่ยวข้องกัน พร้อม access control:

```bash
# สร้าง Knowledge Filter สำหรับทีม HR
curl -X POST http://localhost:8000/knowledge-filter \
  -H "Authorization: Bearer <token>" \
  -d '{
    "name": "HR Policies 2024",
    "description": "นโยบายและระเบียบการทำงานของบริษัท",
    "query_data": "นโยบาย HR กฎระเบียบ การลา สวัสดิการ",
    "allowed_users": [],
    "allowed_groups": ["all_employees"]
  }'

# Response
{
  "id": "kf_hr_policies_2024",
  "name": "HR Policies 2024",
  "owner": "hr@company.com",
  "allowed_groups": ["all_employees"]
}
```

```bash
# สร้าง Knowledge Filter สำหรับ Finance (จำกัด)
curl -X POST http://localhost:8000/knowledge-filter \
  -d '{
    "name": "Financial Reports Q4 2024",
    "description": "รายงานการเงิน Q4 2024",
    "allowed_users": ["ceo@company.com", "cfo@company.com"],
    "allowed_groups": ["finance"]
  }'
```

---

### การค้นหาพร้อม Filter

เมื่อ User ค้นหา ระบบจะ **filter อัตโนมัติ** ตาม identity:

```
User: engineer01@company.com (อยู่ใน group: engineering, all_employees)

คำถาม: "นโยบายการลาคือกี่วัน?"

ระบบ:
  1. Embed คำถาม → vector
  2. KNN Search ใน OpenSearch
  3. Filter อัตโนมัติ:
     WHERE owner = 'engineer01@company.com'
     OR 'engineer01@company.com' IN allowed_users
     OR 'engineering' IN allowed_groups
     OR 'all_employees' IN allowed_groups
  4. ส่งผลลัพธ์ที่ผ่าน filter ไป LLM
  5. LLM ตอบจาก context ที่ได้รับสิทธิ์เท่านั้น

→ เห็นเอกสาร HR ทั่วไป แต่ไม่เห็น Financial Reports
```

---

### ตัวอย่าง: การ Query กับ Role ต่างกัน

```
เอกสารในระบบ:
  A. "นโยบายการลา.pdf"      → allowed_groups: ["all_employees"]
  B. "รายงาน Q4 2024.pdf"   → allowed_groups: ["finance", "executive"]
  C. "IT Security Policy.pdf" → allowed_groups: ["it", "all_employees"]
  D. "Salary Band 2024.pdf"   → allowed_users: ["ceo@co.com", "cfo@co.com"]
                                 allowed_groups: ["hr"]

คำถาม: "อยากรู้เรื่องนโยบายบริษัท"

User A (engineer, all_employees):
  → เห็น: A, C
  → ไม่เห็น: B, D

User B (finance, all_employees):
  → เห็น: A, B, C
  → ไม่เห็น: D

User C (hr, all_employees):
  → เห็น: A, C, D
  → ไม่เห็น: B

User D (ceo@co.com, executive, all_employees):
  → เห็น: A, B, C, D (ทั้งหมด)
```

---

### Mode ที่ 2: No-Auth Mode (ไม่มี OAuth)

ถ้าไม่ตั้งค่า `GOOGLE_OAUTH_CLIENT_ID` → ระบบเป็น **anonymous mode**:

```
ทุกคนที่เข้าถึง URL → ใช้ identity เดียวกัน ("anonymous@localhost")
→ เห็นเอกสารทั้งหมดที่ owner เป็น anonymous
→ ไม่มี user isolation
→ เหมาะสำหรับ: Internal team เล็กๆ, prototype, demo
```

**แนะนำสำหรับองค์กร**: ใช้ OAuth mode เสมอ

---

## 4. ต้องการคนดูแลต่อเนื่องไหม?

### คำตอบ: **ต้องดูแลต่อเนื่อง** — แต่งานส่วนใหญ่ไม่ซับซ้อน

แบ่งเป็น 3 ระดับ:

---

### ระดับที่ 1: งาน Day-to-Day (ทำเป็นประจำ)

| งาน | ความถี่ | ใครทำ | เวลาที่ใช้ |
|-----|---------|-------|------------|
| Upload เอกสารใหม่ | รายวัน/รายสัปดาห์ | Content Owner ของแต่ละแผนก | 5-15 นาที |
| ลบเอกสารที่ outdated | รายเดือน | Admin | 30 นาที |
| ตรวจสอบ Task Queue (ถ้า ingest ล้มเหลว) | รายสัปดาห์ | Admin | 15 นาที |
| ตรวจสอบ chat logs (quality check) | รายสัปดาห์ | Admin | 30 นาที |

---

### ระดับที่ 2: งาน Maintenance (รายเดือน)

| งาน | รายละเอียด | เครื่องมือ |
|-----|------------|-----------|
| **Update เอกสาร** | ลบเก่า → Upload ใหม่ | Web UI หรือ API |
| **ตรวจสอบ OpenSearch health** | Disk usage, index size | OpenSearch Dashboards (port 5601) |
| **Backup ข้อมูล** | Backup `opensearch-data/` volume | Docker volume backup |
| **Review Access Control** | ตรวจสอบว่า permission ยังถูกต้องไหม | API + manual review |
| **Update API Keys** | Rotate keys ที่ไม่ได้ใช้ | Web UI → Settings → API Keys |

---

### ระดับที่ 3: งาน Ongoing Development (ถ้าต้องการ)

งานพวกนี้ทำแค่เมื่อต้องการ ไม่จำเป็นต้องทำทุกเดือน:

| งาน | เมื่อไหร่ต้องทำ | ความยาก |
|-----|----------------|---------|
| ปรับ `system_prompt` | ต้องการเปลี่ยน personality ของ AI | ง่าย |
| ปรับ `chunk_size` / `chunk_overlap` | คำตอบไม่ครบถ้วน/ไม่แม่นยำ | ปานกลาง |
| เพิ่ม connector ใหม่ (Google Drive, etc.) | ต้องการดึงข้อมูลจาก source ใหม่ | ปานกลาง |
| เปลี่ยน LLM Model | มี model ใหม่ที่ดีกว่า | ง่าย |
| ปรับ Langflow flows | ต้องการ custom workflow | ยาก |
| Scale OpenSearch | ข้อมูลมากขึ้น, user มากขึ้น | ยาก |
| Integration กับระบบอื่น | เชื่อมต่อ HR system, Slack, etc. | ยาก |

---

### สิ่งที่ **ไม่ต้องทำ** หลัง setup เสร็จ

```
✓ ไม่ต้อง retrain model (ใช้ pre-trained embeddings)
✓ ไม่ต้อง maintain ML pipeline
✓ ไม่ต้อง tune hyperparameters (default ใช้ได้ดีแล้ว)
✓ ไม่ต้อง monitor GPU (ถ้าใช้ OpenAI/Anthropic)
✓ OpenSearch index ดูแลตัวเองได้ (auto-merge, compaction)
✓ Langflow flows backup อัตโนมัติทุก 5 นาที
```

---

### ทีมที่แนะนำสำหรับองค์กร

```
องค์กรขนาดเล็ก (< 100 คน):
  • 1 IT Admin (part-time) — ดูแล infrastructure
  • Content Owners แต่ละแผนก — Upload/Update เอกสาร

องค์กรขนาดกลาง (100-500 คน):
  • 1 DevOps/IT Admin (full-time)
  • Knowledge Manager 1-2 คน
  • Content Owners แต่ละแผนก

องค์กรขนาดใหญ่ (> 500 คน):
  • DevOps Team
  • AI/ML Engineer (สำหรับ custom development)
  • Knowledge Management Team
```

---

## ตัวอย่างข้อมูล

### ตัวอย่างที่ 1: บริษัทประกัน

```
โครงสร้าง Knowledge Base:
├── [PUBLIC - all_employees]
│   ├── นโยบายบริษัท_2024.pdf
│   ├── Employee_Handbook.pdf
│   └── วันหยุดประจำปี_2024.pdf
│
├── [RESTRICTED - agents, supervisors]
│   ├── Product_Manual_ประกันชีวิต.pdf
│   ├── Product_Manual_ประกันรถ.pdf
│   └── Underwriting_Guidelines.pdf
│
├── [CONFIDENTIAL - management, executives]
│   ├── Loss_Ratio_Report_Q4.pdf
│   └── Pricing_Strategy_2025.pdf
│
└── [HR ONLY - hr]
    ├── Salary_Grade_2024.pdf
    └── Performance_KPI_Template.pdf
```

ตัวอย่างคำถามและสิ่งที่ AI ตอบ:

```
[Agent] "ประกันชีวิตครอบคลุมอะไรบ้าง?"
→ ดึงจาก Product_Manual_ประกันชีวิต.pdf ✓

[HR Staff] "Salary Grade ของ Senior Engineer คือเท่าไหร่?"
→ ดึงจาก Salary_Grade_2024.pdf ✓

[Agent] "Salary Grade ของ Senior Engineer คือเท่าไหร่?"
→ "ไม่พบข้อมูลที่เกี่ยวข้อง" (ไม่มีสิทธิ์เข้าถึง) ✓
```

---

### ตัวอย่างที่ 2: โรงพยาบาล

```
Knowledge Base Structure:
├── [all_staff]
│   ├── Hospital_Policy.pdf
│   ├── Emergency_Protocol.pdf
│   └── Patient_Safety_Guidelines.pdf
│
├── [doctors, nurses]
│   ├── Clinical_Guidelines_2024.pdf
│   ├── Drug_Reference_Manual.pdf
│   └── Procedure_Protocols.pdf
│
├── [pharmacists]
│   ├── Drug_Interaction_Database.pdf
│   └── Dosage_Guidelines.pdf
│
└── [admin, billing]
    ├── Insurance_Claims_Process.pdf
    └── Billing_Codes_2024.pdf
```

---

### ตัวอย่างที่ 3: ตัวอย่าง Query → Response จริง

```
User: nurse01@hospital.com (groups: nurses, all_staff)
คำถาม: "ขนาดยา Amoxicillin สำหรับเด็กอายุ 5 ขวบ คือเท่าไหร่?"

RAG Pipeline:
1. Query embedding: [0.234, -0.456, 0.789, ...]
2. KNN Search → พบ 5 chunks ที่ใกล้เคียง
3. ACL Filter:
   - Drug_Reference_Manual.pdf → allowed_groups: ["doctors", "nurses"] ✓ ผ่าน
   - Drug_Interaction_Database.pdf → allowed_groups: ["pharmacists"] ✗ กรอง
4. ส่ง 5 chunks ที่ผ่าน filter ไป LLM
5. LLM ตอบ:
   "จากคู่มือยา Amoxicillin สำหรับเด็กอายุ 5 ขวบ
   ขนาดยาที่แนะนำคือ 25-50 mg/kg/day แบ่งให้ทุก 8-12 ชั่วโมง
   (Source: Drug_Reference_Manual.pdf, หน้า 45)"
```

---

## Checklist ก่อน Go Live

### Infrastructure
- [ ] เปลี่ยน `OPENSEARCH_PASSWORD` จาก default
- [ ] เปลี่ยน `LANGFLOW_SUPERUSER_PASSWORD` จาก default
- [ ] ตั้งค่า SSL/HTTPS (production)
- [ ] ตั้งค่า domain + reverse proxy (nginx/traefik)
- [ ] ตั้งค่า backup schedule สำหรับ `opensearch-data/`
- [ ] ตั้งค่า monitoring (OpenSearch Dashboards)

### Authentication
- [ ] ตัดสินใจว่าจะใช้ OAuth หรือ no-auth mode
- [ ] ถ้าใช้ OAuth: ตั้งค่า Google/Microsoft app credentials
- [ ] กำหนด group structure ขององค์กร
- [ ] ทดสอบ login flow

### Data
- [ ] วางแผน folder structure และ permission
- [ ] Ingest เอกสาร pilot (5-10 ไฟล์)
- [ ] ทดสอบ access control กับ test users
- [ ] ตรวจสอบ quality ของ search results

### LLM
- [ ] ตั้งค่า API Key
- [ ] ทดสอบ embedding
- [ ] ปรับ `system_prompt` ให้เหมาะกับองค์กร
- [ ] กำหนด budget/rate limit กับ LLM provider

### Operations
- [ ] กำหนดผู้รับผิดชอบ (Admin, Content Owner)
- [ ] สร้าง runbook สำหรับปัญหาที่พบบ่อย
- [ ] ทดสอบ disaster recovery (restore จาก backup)

---

## สรุป

| คำถาม | คำตอบสั้น |
|-------|-----------|
| Start service แล้วใช้ได้เลยไหม? | **ไม่** — ต้องตั้งค่า LLM API Key และ Onboarding ก่อน |
| ต้องนำข้อมูลเข้าด้วยตัวเองไหม? | **ใช่** — ต้อง Upload เอกสารขององค์กรเอง |
| Permission ตาม Role ได้ไหม? | **ได้** — ทำผ่าน `allowed_users` และ `allowed_groups` |
| ต้องการ Developer ไหม? | **ไม่จำเป็น** สำหรับงาน basic — แต่ต้องการ IT Admin |
| ต้องพัฒนาต่อไหม? | **ขึ้นกับความต้องการ** — default พอใช้งานได้ดี |
| ใช้ได้ offline (no internet)? | **ได้** — ถ้าใช้ Ollama แทน OpenAI |

---

*เอกสารนี้สร้างจากการวิเคราะห์ source code ของ OpenRAG โดยตรง*
*อ้างอิง: `/src/services/`, `/src/api/`, `/flows/`, `/docker-compose.yml`*
