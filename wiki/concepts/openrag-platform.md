---
title: "OpenRAG Platform"
type: concept
tags: [openrag, rag, platform, self-hosted, langflow, opensearch, docling, fastapi, sdk]
sources: [openrag_guide.md, phase1-overview.md, phase2-docling.md, phase3-opensearch.md, phase4-langflow.md, phase5-integration.md, openrag-data-ingestion-guide.md, openrag-organization-guide.md, sdk.md, openrag-allowed-groups-google.md, data-update-guide.md, openrag-flows-payload-permission.md]
related: [wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/langflow-visual-workflow.md, wiki/concepts/docling-document-parser.md, wiki/concepts/hybrid-search-bm25-vector.md]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น
OpenRAG เป็น Open-Source RAG Platform ครบวงจรสำหรับองค์กร เน้น 3 จุด: **Modular** (เลือก LLM/VectorDB อิสระ), **Evaluation-first** (built-in metrics), และ **Production-ready** (async task queue, RBAC, multi-tenant)

## อธิบาย
OpenRAG ประกอบด้วย 5 services ทำงานร่วมกัน:
- **Backend (FastAPI)** — API Gateway, Auth (JWT + OAuth), Task Queue, Cloud Connectors
- **Frontend (Next.js)** — Web UI สำหรับ upload, search, chat
- **Langflow** — Visual AI Pipeline Engine (Ingest + Chat flows)
- **OpenSearch** — Vector DB + Full-text + Semantic Search + Row-level Security
- **Docling** — AI Document Parser (PDF/DOCX/PPTX/ภาพ → Markdown)

ต่างจาก LangChain/LlamaIndex ที่เป็น library building blocks — OpenRAG เป็น complete platform พร้อม deploy ได้เลย

## ประเด็นสำคัญ
- **Access Control ระดับ Document**: `owner`, `allowed_users`, `allowed_groups` ต่อทุก chunk — OpenSearch enforce ผ่าน DLS + OIDC
- **Multiple LLM Providers**: OpenAI, Anthropic, IBM WatsonX, Ollama (local/private)
- **Knowledge Filters**: จำกัด search scope ตาม context (เช่น HR Bot เห็นเฉพาะเอกสาร HR)
- **2 Data Flows**: Ingestion (Upload → Parse → Chunk → Embed → Store) และ Query (Question → Embed → KNN → LLM → Answer + Citations)
- **Public REST API v1**: `/v1/chat`, `/v1/search`, `/v1/documents/ingest` ใช้ API Key auth

## ตัวอย่าง / กรณีศึกษา

**Sellsuki Company Knowledge Bot:**
- ก่อน: พนักงานถาม HR → รอ → ได้คำตอบช้า
- หลัง: ถาม Bot → ได้ทันที 24/7 พร้อม citation ชี้ไฟล์ + หน้า
- ROI: ประหยัด HR time ~40 ชั่วโมง/เดือน = ~12,000 บาท/เดือน
- คืนทุน ~9 เดือน (development cost ~100,000 บาท)

```bash
# Setup ด่วน
cp .env.example .env
docker compose up -d
docker run -d -p 5001:5001 quay.io/docling-project/docling-serve:latest

# URLs
Frontend:    http://localhost:3000
Backend API: http://localhost:8080
Langflow:    http://localhost:7860
OpenSearch:  http://localhost:5601
```

### 8 Ingestion Channels

| Channel | เหมาะกับ | Real-time |
|---------|---------|-----------|
| UI Upload (drag & drop) | ข้อมูลน้อย-กลาง | ❌ |
| Server Path (folder mount) | NAS/network drive | ❌ |
| S3 Bucket | AWS archive | ❌ |
| Google Drive | Google Workspace | ✅ Webhook |
| OneDrive | Microsoft 365 | ✅ Webhook |
| SharePoint | Microsoft 365 Business | ✅ Webhook |
| URL / Web (MCP) | Public docs | ❌ |
| REST API (API Key) | ETL, automation | ❌ |

### Document Update Mechanism

- `document_id = SHA256(file content)` → ไฟล์เดิมเนื้อหาเดิม = **skip อัตโนมัติ**
- เนื้อหาเปลี่ยน → **Delete All → Re-index All** (ไม่มี partial update)
- ใช้ชื่อไฟล์เดิมเสมอเมื่อ update → ระบบจัดการให้

### Official SDK

```
OpenRAGClient
├── .chat              → Chat + streaming + conversation
├── .search            → Semantic search
├── .documents         → Ingest + Delete + List
├── .settings          → Config
├── .models            → LLM/embedding model list
└── .knowledgeFilters  → Knowledge filter management
```

ติดตั้ง: `npm install openrag-sdk` / `pip install openrag-sdk`

### Known RBAC Gap: allowed_groups ไม่ถูก enforce

**ปัญหา**: OpenSearch DLS policy ปัจจุบัน:
```yaml
should:
  - term: { owner: "${user.name}" }
  - term: { allowed_users: "${user.name}" }
  - must_not: { exists: { field: owner } }   # public docs
# ❌ ไม่มี: allowed_groups check เลย!
```

Google Drive Connector เก็บ `allowed_groups` ถูกต้องตอน ingest แต่ **ไม่ถูก enforce ตอน query**
เหตุผล: JWT token ไม่มี group membership claims (ต้องใช้ Google Directory API แยก)

**วิธีแก้**: เพิ่ม `groups` claim ใน JWT + อัพเดท DLS policy (ดู openrag-access-control-rbac)

## ความสัมพันธ์กับ concept อื่น
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — OpenRAG implement RAG pattern ครบวงจร
- [[wiki/concepts/langflow-visual-workflow|Langflow]] — AI pipeline engine ใน OpenRAG
- [[wiki/concepts/docling-document-parser|Docling]] — document parser ใน OpenRAG
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search]] — search strategy ที่ OpenSearch ใช้
- [[wiki/concepts/semantic-caching|Semantic Caching]] — ยังขาดอยู่ใน OpenRAG (ดู openrag-cache-analysis)

## แหล่งที่มา
- [[wiki/sources/frameworks/openrag-platform-overview|OpenRAG Platform Overview]]
- [[wiki/sources/frameworks/openrag-integration-scenarios|Integration Scenarios]]
- [[wiki/sources/frameworks/openrag-data-ingestion-channels|Data Ingestion Channels]]
- [[wiki/sources/frameworks/openrag-document-update-guide|Document Update Guide]]
- [[wiki/sources/frameworks/openrag-access-control-rbac|Access Control & RBAC Analysis]]
- [[wiki/sources/frameworks/openrag-flow-payloads|Flow Payloads & Permission Architecture]]
- [[wiki/sources/frameworks/openrag-organization-deployment|Organization Deployment Guide]]
- [[wiki/sources/frameworks/openrag-sdk-reference|SDK Reference]]
