---
title: "OpenRAG — Platform Overview & Architecture"
type: source
source_file: raw/notes/openrag/docs-lean/phase1-overview.md
url: ""
published: 2026-03-17
tags: [openrag, rag, platform, architecture, langflow, opensearch, docling]
related: [wiki/concepts/openrag-platform.md, wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/langflow-visual-workflow.md, wiki/concepts/docling-document-parser.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/openrag/docs-lean/phase1-overview.md|Phase 1 Overview]]
> Additional sources: [[../../raw/notes/openrag/openrag_guide.md|OpenRAG Guide]], [[../../raw/notes/openrag/docs-lean/README.md|README]]

## สรุป
OpenRAG เป็น Open-Source RAG platform ครบวงจรสำหรับองค์กร ประกอบด้วย 5 services หลักทำงานร่วมกัน: Frontend, Backend (FastAPI), Langflow (AI pipeline), OpenSearch (Vector DB), และ Docling (Document Parser)

## ประเด็นสำคัญ

### Services ทั้งหมด
| Service | บทบาท | Port |
|---------|--------|------|
| **Frontend** (Next.js) | Web UI — upload, search, chat | 3000 |
| **Backend** (FastAPI) | API Gateway, Auth, Task Queue | 8080 |
| **Langflow** | Visual AI Pipeline (Ingest + Chat) | 7860 |
| **OpenSearch** | Vector DB + Semantic Search | 9200 |
| **Docling** | AI Document Parser (PDF/DOCX/Image) | 5001 |
| **OS Dashboards** | OpenSearch Monitor UI | 5601 |

### 2 Data Flows หลัก

**Ingestion Flow:**
```
User Upload → Backend → Langflow → Docling (parse) → Text Splitter → Embedding → OpenSearch
```

**Chat/Query Flow:**
```
User Question → Backend → Langflow → Embed Query → OpenSearch KNN → LLM → Response + Citations
```

### ฟีเจอร์สำคัญ
- **Document-level Access Control**: `owner`, `allowed_users`, `allowed_groups` ต่อทุก document chunk
- **Multiple LLM Providers**: OpenAI, Anthropic Claude, IBM WatsonX, Ollama (local)
- **Multiple Embedding Models**: text-embedding-3-small (default 1536 dim), text-embedding-3-large (3072 dim), Ollama
- **Connectors**: Google Drive, SharePoint, OneDrive, URL ingestion
- **Public REST API v1**: Authenticate ด้วย API Key

### OpenRAG vs LangChain vs LlamaIndex

| Feature | OpenRAG | LlamaIndex | LangChain |
|---------|---------|------------|-----------|
| RAG Pipeline | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Evaluation built-in | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| Agent / Tool use | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Lines of code (basic RAG) | ~30 | ~50 | ~80 |
| Focus | RAG + Eval + Production | Data Indexing | Swiss Army Knife |

**เลือกใช้เมื่อ:**
- ต้องการ RAG ที่วัดผลได้ + deploy เร็ว → OpenRAG
- ต้องการ index data หลากหลายรูปแบบ → LlamaIndex
- ต้องการ Agent + Tools ซับซ้อน → LangChain

### Setup เบื้องต้น
```bash
cp .env.example .env
# ต้องตั้งค่า: OPENSEARCH_PASSWORD, SECRET_KEY, OPENAI_API_KEY
docker compose up -d
docker run -d -p 5001:5001 quay.io/docling-project/docling-serve:latest
```

### Use Case: Sellsuki Company Knowledge Bot
- HR ถามนโยบาย → Bot ตอบได้ทันที 24/7
- WiFi/IT → ถาม Slack Bot
- Dev → ถามใน Cursor ผ่าน MCP integration
- คาดว่าประหยัด HR time ~40 ชั่วโมง/เดือน = ~12,000 บาท/เดือน

## ข้อมูล / หลักฐาน ที่น่าสนใจ
- โครงสร้างไฟล์: `src/main.py` (988 lines), `src/agent.py` (845 lines), `src/config/settings.py` (871 lines)
- Flows directory: `flows/ingestion_flow.json`, `flows/openrag_agent.json`, `flows/openrag_url_mcp.json`, `flows/openrag_nudges.json`
- Backend มี 28 API endpoints ครอบคลุม Chat, Documents, Tasks, Search, Auth, Flows

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว
- OpenRAG เป็น framework สำหรับ self-hosted RAG แบบ complete platform ต่างจาก LangChain/LlamaIndex ที่เป็น library building blocks

## Concepts ที่เกี่ยวข้อง
- [[wiki/concepts/openrag-platform.md|OpenRAG Platform]]
- [[wiki/concepts/rag-retrieval-augmented-generation.md|RAG]]
- [[wiki/concepts/langflow-visual-workflow.md|Langflow]]
- [[wiki/concepts/docling-document-parser.md|Docling]]
