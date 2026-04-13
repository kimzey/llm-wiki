# OpenRAG Learning Docs — Index

เอกสารเรียนรู้ครบชุดสำหรับทำความเข้าใจและใช้งาน OpenRAG

---

## อ่านตามลำดับ (แนะนำ)

| # | ไฟล์ | เนื้อหา | เหมาะสำหรับ |
|---|------|---------|------------|
| 1 | [phase1-overview.md](./phase1-overview.md) | RAG คืออะไร, Architecture, Data Flow | ทุกคน (เริ่มต้น) |
| 2 | [phase2-docling.md](./phase2-docling.md) | Docling: Document Parser, OCR, ข้อจำกัด | ทุกคน |
| 3 | [phase3-opensearch.md](./phase3-opensearch.md) | OpenSearch: Vector DB, KNN, Access Control | Developer / Admin |
| 4 | [phase4-langflow.md](./phase4-langflow.md) | Langflow: Workflow, Flows, Pipeline | Developer |
| 5 | [phase5-integration.md](./phase5-integration.md) | Integration ทุก Service + Scenarios จริง | Developer |

---

## คู่มือปฏิบัติ (อ่านเมื่อต้องการ)

| ไฟล์ | เนื้อหา | เหมาะสำหรับ |
|------|---------|------------|
| [org-rag-guide.md](./org-rag-guide.md) | Setup → Ingest → Chat → API → Access Control | ทุกคนที่จะ deploy |
| [advanced-config.md](./advanced-config.md) | config.yaml, System Prompt, Feature Flags, S3 | Admin / Developer |
| [cost-and-models.md](./cost-and-models.md) | ค่าใช้จ่าย, เปรียบเทียบ Models, Ollama Setup | ผู้ตัดสินใจ / Admin |
| [document-best-practices.md](./document-best-practices.md) | เตรียมเอกสาร, Chunk Strategy, ทดสอบ RAG | ทุกคน |

---

## Quick Reference

### Setup ด่วน
```bash
cp .env.example .env
# แก้: OPENSEARCH_PASSWORD, SECRET_KEY, OPENAI_API_KEY
docker compose up -d
docker run -d -p 5001:5001 quay.io/docling-project/docling-serve:latest
```

### URLs หลัก
```
Frontend:    http://localhost:3000
Backend API: http://localhost:8080
API Docs:    http://localhost:8080/docs
Langflow:    http://localhost:7860
OpenSearch:  http://localhost:5601
```

### Ingest เอกสาร
```bash
curl -X POST "http://localhost:8080/v1/documents/ingest" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -F "file=@document.pdf" \
  -F "allowed_groups=all-employees"
```

### Chat API
```bash
curl -X POST "http://localhost:8080/v1/chat" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{"message": "คำถาม", "session_id": "sess-001"}'
```

### ตรวจ Health
```bash
curl http://localhost:8080/health
curl http://localhost:5001/health
```

---

## Services Summary

| Service | บทบาท | Port |
|---------|--------|------|
| **Docling** | AI Document Parser (PDF/DOCX/Image) | 5001 |
| **Langflow** | Visual Pipeline (Ingest + Chat flows) | 7860 |
| **OpenSearch** | Vector DB + Semantic Search | 9200 |
| **Backend** | FastAPI — API Gateway | 8080 |
| **Frontend** | Web UI (Next.js) | 3000 |
| **OS Dashboards** | OpenSearch Monitor UI | 5601 |

---

## Access Control ย่อ

```
ทุก document มี:
  owner:           "user@co.com"          ← เจ้าของ
  allowed_users:   ["a@co.com"]           ← user เฉพาะ
  allowed_groups:  ["hr-team"]            ← group

User เห็นเอกสารถ้า: เป็น owner OR อยู่ใน allowed_users OR อยู่ใน allowed_groups
Filter ทำงานอัตโนมัติ ไม่ต้องทำอะไรเพิ่ม
```

---

## เลือก Model ย่อ

| สถานการณ์ | LLM | Embedding |
|----------|-----|-----------|
| เริ่มต้น / ถูก | gpt-4o-mini | text-embedding-3-small |
| คุณภาพสูง | gpt-4o | text-embedding-3-small |
| Privacy สูงสุด | Ollama llama3.1 | nomic-embed-text |
| Enterprise | WatsonX granite | ibm/slate-125m |
