---
title: "OpenRAG — Langflow Visual Workflow Engine"
type: source
source_file: raw/notes/openrag/docs-lean/phase4-langflow.md
url: ""
published: 2026-03-17
tags: [langflow, workflow, visual, pipeline, rag, openrag]
related: [wiki/concepts/langflow-visual-workflow.md, wiki/concepts/openrag-platform.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/openrag/docs-lean/phase4-langflow.md|Phase 4 — Langflow]]
> Additional source: [[../../raw/notes/openrag/docs-lean/openrag-backend-vs-langflow.md|Backend vs Langflow]]

## สรุป
Langflow เป็น Open-Source Visual Workflow Builder ที่ OpenRAG ใช้เป็น AI Pipeline Engine — จัดการทุกอย่างที่เกี่ยวกับ AI processing (parse, split, embed, retrieve, generate) ผ่าน Drag-and-Drop UI โดยไม่ต้องแก้โค้ด

## ประเด็นสำคัญ

### Flows หลัก 4 ตัวใน OpenRAG
```
flows/
├── ingestion_flow.json      # Document Ingestion Pipeline
├── openrag_agent.json       # Chat/RAG Agent
├── openrag_url_mcp.json     # URL Ingestion
└── openrag_nudges.json      # Contextual Suggestions
```

### Flow 1: Ingestion Flow
```
[File Input] → [Docling (parse)] → [Split Text (chunk)] → [Embedding] → [OpenSearch Store]
  - chunk_size: 1000, overlap: 200
  - Metadata injected via headers: owner, filename, allowed_groups
```

### Flow 2: RAG Agent Flow (Chat)
```
[Chat Input] → [OpenSearch Retriever (KNN)] → [Context Builder] → [LLM] → [Chat Output]
  - JWT ส่งผ่าน headers → OpenSearch apply DLS filter อัตโนมัติ
  - Streaming response รองรับ
```

### Backend vs Langflow: แบ่งหน้าที่ชัดเจน
| ฟีเจอร์ | Backend | Langflow |
|---------|---------|----------|
| Authentication & JWT | ✅ | ❌ |
| Task Queue & Async | ✅ | ❌ |
| Cloud Connectors | ✅ | ❌ |
| Settings Management | ✅ | ❌ |
| Document Parse (Docling) | ❌ | ✅ |
| Text Splitting | ❌ | ✅ |
| Embedding | ❌ | ✅ |
| Vector Index Write | ❌ | ✅ |
| LLM Call / Agent | ❌ | ✅ |
| DLS Security | ❌ | ❌ (OpenSearch เอง) |

**กฎหลัก:**
- Backend = Infrastructure + Security + Orchestration ("ใคร", "จัดการ", "ปลอดภัย")
- Langflow = AI Processing Pipeline ("ประมวลผลเอกสาร", "AI reasoning", "ปรับแต่งได้")

### วิธีที่ Backend เรียก Langflow
```python
# 1. Upload file ไปที่ Langflow Files API
POST /api/v1/files/upload/{flow_id}

# 2. Run Ingestion Flow พร้อม tweaks
POST /api/v1/run/{ingest_flow_id}
Body: { "input_value": file_path, "tweaks": { "DoclingComponent": {...}, "OpenSearchStore": {...} } }

# 3. Chat (Streaming)
POST /api/v1/run/{chat_flow_id}?stream=true
Headers: { JWT, embedding_model, filter }
```

### Langflow Components ที่ใช้
| Component | หน้าที่ |
|-----------|---------|
| DoclingFileLoader | เรียก Docling API parse เอกสาร |
| RecursiveCharacterTextSplitter | แบ่ง text เป็น chunks |
| OpenAIEmbeddings / OllamaEmbeddings | สร้าง embeddings |
| OpenSearchVectorStore | บันทึก/ค้นหาใน OpenSearch |
| ChatOpenAI / ChatAnthropic | LLM component |
| Prompt | System prompt template |

### Auto-initialization เมื่อ Backend Start
1. รอ Langflow พร้อม (health check)
2. สร้าง Langflow API key อัตโนมัติ
3. โหลด Flows จาก `/flows/` directory
4. บันทึก Flow IDs ลง `.env`
5. Periodic backup ทุก 5 นาที

### Timeouts ที่สำคัญ
```bash
LANGFLOW_TIMEOUT=2400        # 40 นาที — รอ flow ประมวลผล
LANGFLOW_CONNECT_TIMEOUT=30  # 30 วินาที
INGESTION_TIMEOUT=3600       # 60 นาที — timeout สำหรับ ingestion task
```

## ข้อมูล / หลักฐาน ที่น่าสนใจ
- Langflow ไม่รู้จัก user — มันแค่รับ metadata จาก headers ที่ Backend inject
- Backend มี Direct Agent mode (agent.py) ที่ไม่ผ่าน Langflow สำหรับ `/chat` endpoint
- ปรับ flow ใน Langflow UI ได้โดยไม่ต้อง restart หรือแก้โค้ด

## Concepts ที่เกี่ยวข้อง
- [[wiki/concepts/langflow-visual-workflow|Langflow Visual Workflow]]
- [[wiki/concepts/openrag-platform|OpenRAG Platform]]
