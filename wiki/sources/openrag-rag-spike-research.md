---
title: "RAG Spike Research — Vector DB, Chunking, pgvector vs OpenSearch"
type: source
source_file: raw/notes/openrag/docs-lean/spike-rag-research-summary.md
url: ""
published: 2026-03-18
tags: [rag, spike, pgvector, opensearch, chunking, vector-db, research]
related: [wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/hybrid-search-bm25-vector.md, wiki/concepts/rag-chunking-strategies.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/openrag/docs-lean/spike-rag-research-summary.md|Spike RAG Research Summary]]

## สรุป
Spike card วันที่ 2026-03-18 — ศึกษา Vector Database สำหรับระบบ Q&A อิงเอกสาร (RAG) เปรียบเทียบ pgvector vs OpenSearch vs Qdrant พร้อม findings จากการศึกษา OpenRAG source code จริง

## ประเด็นสำคัญ

### Metadata Schema ที่แนะนำสำหรับ RAG Chunks
```json
{
  "text": "...chunk content...",
  "embedding": [0.12, -0.45, ...],
  "document_id": "abc123",
  "filename": "hr-policy-2026.pdf",
  "page": 3,
  "owner": "user-id",
  "allowed_users": ["alice", "bob"],
  "allowed_groups": ["hr-team"],
  "indexed_time": "2026-03-18T10:00:00",
  "embedding_model": "text-embedding-3-small"
}
```
**ข้อสำคัญ:** ออกแบบ metadata schema ให้ดีตั้งแต่ต้น เพราะ re-index เพื่อเปลี่ยน schema ต้องทำทั้งหมด

### Chunking Strategy เปรียบเทียบ
| Strategy | ใช้ | ข้อดี | ข้อเสีย |
|----------|-----|-------|---------|
| **Fixed-size** (OpenRAG default) | chunk=1000, overlap=200 | เร็ว ง่าย คาดเดาได้ | อาจตัดกลางความหมาย |
| **Recursive** | `\n\n` → `\n` → `. ` → ` ` | natural boundary | ซับซ้อนกว่า |
| **Semantic** | วิเคราะห์ topic shift | สมบูรณ์ที่สุด | ช้า + แพง |
| **Structural** | 1 chunk = 1 page/section | ไม่ตัดกลางเรื่อง | chunk size ไม่สม่ำเสมอ |

**Rule of thumb:** เริ่มที่ 1000/200 แล้ว tune ตาม retrieval quality จริงๆ

### pgvector vs OpenSearch vs Qdrant
| | pgvector | OpenSearch | Qdrant |
|---|---------|-----------|--------|
| Setup | ใช้ Postgres ที่มีอยู่ | ต้องติดตั้งแยก | service แยก |
| Hybrid Search | ทำเองผ่าน SQL | built-in | built-in |
| Access Control | SQL WHERE | Row-level Security | Filter objects |
| Ops Complexity | ต่ำ (ทีมรู้ Postgres) | กลาง | ต่ำ-กลาง |
| Use Case | ข้อมูลน้อย-กลาง | เอกสารเยอะ + full-text | vector-first |

**เลือก pgvector เมื่อ:** ทีมใช้ Postgres อยู่แล้ว, ข้อมูล < 10M docs, ต้องการ transactional consistency
**เลือก OpenSearch เมื่อ:** ต้องการ hybrid search built-in, scale ใหญ่, full-text search สำคัญ

### Findings จาก OpenRAG Source Code (จริง)
| ประเด็น | สิ่งที่พบ |
|---------|---------|
| Vector DB | ใช้ **OpenSearch** ไม่ใช่ pgvector |
| Chunker | `CharacterTextSplitter` — separator `"\n"`, size 1000, overlap 200 |
| Doc Parser | **Docling Serve** — รองรับ PDF/DOCX/PPT/MD/Images |
| Embedding | multi-model — `text-embedding-3-small` default |
| Hybrid Search | Vector + BM25 keyword ใน OpenSearch |

### Q&A Flow ภาพรวม
```
Ingestion: Doc → Parser → Clean → Chunk → Embed → VectorDB
Query:     Question → Embed → VectorSearch → Context → LLM → Answer
Hybrid:    Query → [VectorSearch + KeywordSearch] → RRF Rank Fusion → LLM
```

### สิ่งที่ต้องตัดสินใจก่อน Implement
- [ ] Embedding Model: OpenAI (cloud) หรือ Ollama (on-premise)?
- [ ] Chunk Strategy: Fixed-size 1000/200 หรือ Recursive?
- [ ] Vector DB: pgvector บน Postgres เดิม หรือ service แยก?
- [ ] Access Control: ทุกคนเห็นทั้งหมด หรือ per-user/per-team filter?
- [ ] Re-ingestion strategy: re-index ทั้งหมด หรือ upsert เฉพาะที่เปลี่ยน?

## ข้อมูล / หลักฐาน ที่น่าสนใจ
- Chunk size ที่ดีต้องทดสอบกับ data จริง — ไม่มี one-size-fits-all
- pgvector รองรับ cosine similarity: `embedding <=> $1` (cosine), `<->` (L2)
- Hybrid Search ด้วย pgvector ต้องเขียน SQL เอง รวม `tsvector` กับ vector search

## Concepts ที่เกี่ยวข้อง
- [[wiki/concepts/rag-retrieval-augmented-generation.md|RAG]]
- [[wiki/concepts/hybrid-search-bm25-vector.md|Hybrid Search]]
- [[wiki/concepts/rag-chunking-strategies.md|RAG Chunking Strategies]]
