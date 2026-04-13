---
title: "Arona Deep Dive — สถาปัตยกรรมและ Ingestion Pipeline"
type: source
source_file: raw/notes/arona/deepdive/INDEX.md
url: ""
published: 2026-04-13
tags: [arona, architecture, ingestion-pipeline, clustering, bun, paradedb]
related: [wiki/concepts/rag-retrieval-augmented-generation.md, wiki/sources/arona-overview.md, wiki/concepts/hybrid-search-bm25-vector.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/arona/deepdive/INDEX.md|deepdive/INDEX]] | [[../../raw/notes/arona/deepdive/01-architecture-and-lifecycle.md|01]] | [[../../raw/notes/arona/deepdive/02-ingestion-pipeline.md|02]] | [[../../raw/notes/arona/deepdive/03-retrieval-and-ai.md|03]] | [[../../raw/notes/arona/deepdive/04-security-and-maintenance.md|04]]

## สรุป

ชุดบทเรียนเจาะลึก 4 บท สำหรับทำความเข้าใจ Arona RAG อย่างถ่องแท้ ครอบคลุมตั้งแต่ architecture, ingestion pipeline, retrieval & AI engine ไปจนถึง security และ maintenance

## ประเด็นสำคัญ

### โครงสร้างบทเรียน

**Lesson 1: Architecture & Lifecycle**
- ปรัชญาการออกแบบ: Bun, Elysia, ParadeDB
- `src/index.ts` — Entry point: clustering (CPU-1 workers), cron 12 ชั่วโมง
- `src/server.ts` — CORS, OpenAPI, OTel tracing, cookie security
- Query trace ตั้งแต่ต้นจนจบ

**Lesson 2: Ingestion Pipeline**
- Extract: Clone/read documentation repo
- Chunk: ตัดตาม `## headers`
- Plan: เช็คกับ DB ว่าเนื้อหาเปลี่ยนไหม (ประหยัด embedding cost)
- Embed: OpenAI text-embedding-3-small, 1536 dimensions, 3 embeddings/chunk
- Save: Insert/Update ใน ParadeDB

**Lesson 3: Retrieval & AI Engine**
- Hybrid Search SQL ใน query เดียว (BM25 + Vector)
- AI Tools และ Agentic Loop ด้วย Vercel AI SDK
- Context management + Conversation History (compress → ใช้ 3 ล่าสุด)
- Cache 3 ชั้น (LRU → Semantic → Redis)

**Lesson 4: Security & Maintenance**
- Auth + Role-based access (สำหรับ Sellsuki version)
- `.env` configuration + Axiom monitoring
- Knowledge base maintenance + Re-indexing
- Docker deployment + Future development

### Ingestion Planning (Diff-aware)

```
1. Hash เนื้อหาแต่ละ chunk
2. เปรียบเทียบกับ DB
   ├── ไม่เปลี่ยน → ข้าม (ประหยัด embedding cost)
   ├── เปลี่ยนน้อย (similarity > 98%) → update metadata เท่านั้น
   └── เปลี่ยนเยอะ → embed ใหม่ + update
3. ลบ chunk ที่ไม่มีแล้ว
```

### โครงสร้าง Source Code
```
src/
├── index.ts          # Clustering + Cron
├── server.ts         # Elysia server
├── libs/             # Core utilities
│   ├── ai.ts        # AI providers
│   ├── cache.ts     # Cache wrapper
│   ├── database.ts  # DB connection
│   ├── embedding.ts # Embedding generation
│   ├── structure.ts # Indexing system
│   └── retry.ts     # Retry mechanism
└── modules/
    ├── ai/          # RAG logic
    │   ├── service.ts  # ask(), search(), readPage()
    │   └── libs/
    │       ├── tool.ts         # AI tools definition
    │       └── semantic-cache.ts
    └── pow/         # Proof of Work
```

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG (Retrieval-Augmented Generation)]]
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search: BM25 + Vector]]
- [[wiki/sources/arona-overview|Arona System Overview]]
