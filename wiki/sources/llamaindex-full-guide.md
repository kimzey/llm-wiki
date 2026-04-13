---
title: "LlamaIndex Full Guide — Sellsuki RAG System"
type: source
source_file: raw/notes/tool-rag/llamaindex-full-guide.md
url: ""
published: 2026-01-01
tags: [llamaindex, rag, fastapi, docker, typescript, sellsuki, production, checklist]
related: [wiki/concepts/llamaindex-framework.md, wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/rag-chunking-strategies.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/tool-rag/llamaindex-full-guide.md|Original file]]

## สรุป

Full guide สำหรับ Sellsuki RAG System: Checklist ครบทุก Phase (Infrastructure → Ingestion → Query → Cache → API → Interface → Observability), Deliverables ที่ควรได้, Server Setup (FastAPI + Docker Compose), TypeScript vs Python comparison, และ Full Architecture แบบละเอียด

## ประเด็นสำคัญ

### 7-Phase Checklist

| Phase | หัวข้อหลัก |
|-------|-----------|
| 1. Infrastructure | ParadeDB + Dragonfly setup + DB schema |
| 2. Ingestion Pipeline | Outline API reader, Chunking, Metadata schema, IngestionCache, Incremental update |
| 3. Query Pipeline | VectorStoreIndex, Hybrid Search, Reranker, Metadata Filter/Permission |
| 4. Caching Layer | Embedding Cache + Semantic Cache + TTL strategy |
| 5. API Server | FastAPI: POST /query, /ingest, WS /stream + Auth + Rate limiting |
| 6. Interface | Web Chat, LINE Bot, Slack Bot, Cursor MCP |
| 7. Observability | Logging, Tracing (LlamaTrace), Dashboard, Alerts |

### Deliverables ที่ได้เมื่อทำเสร็จ

- ✅ ระบบตอบคำถามภาษาไทย/อังกฤษ 24/7 พร้อม source citation
- ✅ Permission-aware: พนักงานทั่วไป vs HR vs Dev เห็น doc ต่างกัน
- ✅ อัปเดตเอกสาร → ระบบรู้อัตโนมัติ
- ✅ Semantic cache < 100ms สำหรับคำถามซ้ำ
- ✅ Admin ดูได้ว่าพนักงานถามอะไร ตอบถูกไหม

### ต้องมี Server ไหม?

ใช่ ต้องมี:
```
LINE/Slack/Web → ต้องการ HTTPS endpoint
LlamaIndex = Python library ไม่ใช่ service → ต้องมี API server ห่อ
```

**Minimal Docker Compose:**
```yaml
services:
  api:     # FastAPI
  paradedb: # image: paradedb/paradedb:latest
  dragonfly: # image: docker.dragonflydb.io/dragonflydb/dragonfly:latest
```

### TypeScript ได้ไหม?

LlamaIndex มี 2 version แต่ Python feature ครบกว่ามาก:

| Feature | Python | TypeScript |
|---------|--------|------------|
| IngestionPipeline + Cache | ✅ built-in | ❌ ต้องเขียนเอง |
| SemanticSplitter | ✅ | ❌ |
| HierarchicalNodeParser | ✅ | ❌ |
| QuestionsAnsweredExtractor | ✅ | ❌ |
| PGVectorStore | ✅ | ✅ |
| Streaming | ✅ | ✅ |

**แนะนำสำหรับทีม TypeScript:**
```
Option B: Python FastAPI (RAG Core) ← LlamaIndex Python
        + TypeScript Next.js (UI/API Layer)
```

### Full Architecture (8-step Query Flow)

```
User Question
  → Semantic Cache Check (Dragonfly)  → HIT: return
  → Embed question (+ Embedding cache)
  → Metadata filter (Permission: access_level, department)
  → Hybrid Retrieve (ParadeDB: HNSW + BM25 RRF) → top 10
  → Rerank (cross-encoder / Cohere) → top 3
  → Build prompt + context
  → LLM (GPT-4o) streaming
  → Return {answer, sources, confidence} + store to cache
```

### Hosting Options

| Option | เหมาะกับ | ราคาประมาณ |
|--------|---------|-----------|
| Docker Compose (on-prem) | Dev / เริ่มต้น | เซิร์ฟเวอร์ที่มี |
| Railway / Render | MVP เร็ว | $20-50/เดือน |
| AWS ECS / GCP Cloud Run | Production | $50-200/เดือน |
| Kubernetes | Scale ใหญ่ | มี DevOps แล้ว |

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- LlamaIndex Python `IngestionCache` + `RedisKVStore` (Dragonfly): สำคัญมาก ไม่มีใน TypeScript version ต้องเขียนเองทั้งหมด
- `Document.id_` ใน LlamaIndex: ต้องกำหนดเสมอเพื่อให้ update/delete ตาม doc ได้ถูกต้อง
- `excluded_llm_metadata_keys`: ประหยัด tokens โดยซ่อน metadata ที่ไม่เกี่ยวข้องจาก LLM
- LlamaIndex Node มี `NodeRelationship`: PREVIOUS, NEXT, PARENT, CHILD, SOURCE — ช่วย hierarchical retrieval

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/llamaindex-framework|LlamaIndex Framework]]
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG — Retrieval-Augmented Generation]]
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]]
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search — BM25 + Vector]]
