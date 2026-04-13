---
title: "OpenRAG — Integration Scenarios & Full Stack"
type: source
source_file: raw/notes/openrag/docs-lean/phase5-integration.md
url: ""
published: 2026-03-17
tags: [openrag, integration, api, access-control, knowledge-filter, google-drive]
related: [wiki/concepts/openrag-platform.md, wiki/concepts/rag-retrieval-augmented-generation.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/openrag/docs-lean/phase5-integration.md|Phase 5 — Integration]]

## สรุป
ครอบคลุม Scenarios จริงของการใช้ OpenRAG: อัปโหลดเอกสาร HR, ถามคำถาม, ใช้ Public API, Knowledge Filters, และ Google Drive Connector — พร้อม step-by-step trace ของแต่ละ request

## ประเด็นสำคัญ

### Scenario 1: Ingest HR Policy (ตัวอย่างสมบูรณ์)
```
Frontend → Backend (create task) → Langflow upload → Langflow run flow
→ Docling parse (8.3s) → 62 chunks → embed (62 API calls) → OpenSearch index
→ task status: "success", chunks_indexed: 62
```
Metadata ที่ถูก inject:
```json
{ "owner": "hr.manager@company.com", "allowed_groups": ["all-employees"] }
```

### Scenario 2: Employee Chat (ตัวอย่างสมบูรณ์)
```
User: "ถ้าป่วย 5 วันต้องทำอย่างไร?"
→ embed query → KNN search (filter by user groups)
→ Top 3 chunks (score 0.943, 0.891, 0.834)
→ LLM generate response:
  "1. แจ้งหัวหน้างานวันแรก
   2. ใบรับรองแพทย์ (ลาเกิน 3 วัน) ส่งภายใน 7 วัน
   3. สิทธิลาป่วย 30 วัน/ปี
   📄 HR Policy 2024 หน้า 3-4"
```

### Scenario 3: Public API v1
```bash
# Chat
POST /v1/chat  -H "Authorization: Bearer sk-openrag-xxx"
# Ingest
POST /v1/documents/ingest  -F "file=@doc.pdf" -F "allowed_groups=all-employees"
```

### Scenario 4: Knowledge Filters
```bash
# สร้าง filter สำหรับ HR Bot
POST /knowledge-filter
{ "name": "HR Documents", "filter": { "terms": { "allowed_groups": ["hr-team", "all-employees"] } } }

# ใช้ filter ใน chat
{ "message": "ลาป่วยได้กี่วัน?", "knowledge_filter_id": "kf-hr-001" }
```

### Scenario 5: Google Drive Connector
```
1. OAuth connect → เลือก folder
2. POST /connectors/google_drive/sync → download → ingest ทุกไฟล์
3. Webhook: เมื่อไฟล์เปลี่ยน → re-index อัตโนมัติ
4. ACL extraction จาก Google permissions → allowed_users/allowed_groups
```

### API Endpoints ครบทุกอัน

**Chat:** `POST /chat`, `POST /langflow`, `GET /chat/history`
**Documents:** `POST /upload`, `POST /upload_ingest_router`, `POST /documents/delete-by-filename`
**Tasks:** `GET /tasks`, `GET /tasks/{id}`, `POST /tasks/{id}/cancel`
**Knowledge Filters:** `POST/GET/PUT/DELETE /knowledge-filter`
**Connectors:** `POST /connectors/{type}/sync`, `GET /connectors/{type}/status`
**Public API v1:** `POST /v1/chat`, `POST /v1/search`, `POST /v1/documents/ingest`
**Health:** `GET /health`, `GET /docling/health`

### Troubleshooting ที่พบบ่อย
| ปัญหา | สาเหตุ | วิธีแก้ |
|-------|--------|---------|
| Docling ไม่ตอบสนอง | Service not running | `docker run -p 5001:5001 docling-serve` |
| OpenSearch ไม่ start | Password ไม่ซับซ้อน | ต้องมี Upper, Lower, Number, Special char |
| Vector dimension mismatch | เปลี่ยน embedding model | ลบ index + re-index ทั้งหมด |
| Ingestion timeout | ไฟล์ขนาดใหญ่ | เพิ่ม INGESTION_TIMEOUT=7200 |

## ข้อมูล / หลักฐาน ที่น่าสนใจ
- Task status: pending → processing → success/failed/cancelled
- `UPLOAD_BATCH_SIZE=25` — process 25 ไฟล์พร้อมกัน
- OpenSearch requires `vm.max_map_count=262144`
- Langflow Flow ID เก็บใน .env และ auto-update เมื่อ import flow ใหม่

## Concepts ที่เกี่ยวข้อง
- [[wiki/concepts/openrag-platform.md|OpenRAG Platform]]
- [[wiki/concepts/rag-retrieval-augmented-generation.md|RAG]]
