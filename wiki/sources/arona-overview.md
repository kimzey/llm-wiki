---
title: "Arona — ระบบ RAG สำหรับ Elysia Documentation"
type: source
source_file: raw/notes/arona/README.md
url: ""
published: 2026-04-13
tags: [arona, rag, elysia, self-hosted, bun, typescript]
related: [wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/hybrid-search-bm25-vector.md, wiki/concepts/semantic-caching.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/arona/README.md|README]] | [[../../raw/notes/arona/ARONA_GUIDE.md|ARONA_GUIDE]]

## สรุป

Arona คือระบบ RAG (Retrieval-Augmented Generation) แบบ self-hosted สำหรับค้นหาและตอบคำถามจากเอกสารของ Elysia Framework สร้างขึ้นเพราะ RaaS (RAG as a Service) เช่น Kapa, Mintlify, Mendable มีราคาสูงถึง $250-1,000/เดือน ในขณะที่ Arona ทำงานได้ในราคาเพียง ~$21/เดือน สำหรับ 750 request/วัน

## ประเด็นสำคัญ

### แรงจูงใจในการสร้าง
- RaaS providers แพงเกินไป (Kapa ~$1,000, Mintlify ~$250/เดือน)
- มีเอกสารบน VitePress อยู่แล้ว ไม่ต้องการย้าย
- ต้องการ control 100% ไม่ติด vendor lock-in

### Tech Stack
| Component | Technology | หน้าที่ |
|-----------|-------------|----------|
| Backend | Elysia (Bun) | API Server |
| Database | ParadeDB | BM25 + Vector Search |
| Cache | DragonflyDB | Redis-compatible |
| AI Main | GPT OSS 120B | ตอบคำถาม |
| AI Small | GPT OSS 20B | ปรับคำถาม (normalize) |
| Embedding | OpenAI text-embedding-3-small | Vector 1536 มิติ |
| Monitoring | Axiom | Logging + Tracing (OTel) |
| Security | Cloudflare Turnstile + PoW | ป้องกันบอท |

### Request Flow (7 ขั้นตอน)
1. **Security** — Proof of Work 19 bits + Cloudflare Turnstile
2. **Cache Check** — Redis + Semantic Cache + LRU (ถ้าเจอ ตอบทันที)
3. **Query Normalization** — ลบ filler words ด้วย small model
4. **AI Tool Calling** — AI เลือกใช้ tools: `search`, `readPage`, `tableOfContents`
5. **Hybrid Search** — BM25 + Vector ในฐานข้อมูลเดียวกัน
6. **AI Generation** — stream คำตอบกลับ
7. **Cache Store** — เก็บลง cache เผื่อครั้งหน้า

### Caching Strategy (3 ชั้น)
| Layer | Storage | TTL | วัตถุประสงค์ |
|-------|---------|-----|-------------|
| In-Memory LRU | RAM | 30s–9hr | Burst cache |
| Redis | DragonflyDB | 3hr–10hr | Persistent |
| Semantic Cache | Redis Vector | 4hr | Similarity ≥ 90% |

### Database Schema
```sql
CREATE TABLE documents (
    link VARCHAR(255) PRIMARY KEY,
    file VARCHAR(255) NOT NULL,
    title VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    summary TEXT NOT NULL,
    weight FLOAT NOT NULL DEFAULT 0.5,
    embedding VECTOR(1536),
    title_embedding VECTOR(1536),
    file_name_embedding VECTOR(1536),
    sequence smallint NOT NULL DEFAULT 0
);
```

### Document Weight System
```
essential: 1.0  →  blog: 0.8  →  eden: 0.7  →  patterns: 0.5
unknown: 0.4  →  migrate/integrations/tutorial: 0.3
```

### Vector Scoring Formula
```
score = 0.1 × title_similarity + 0.675 × content_similarity
      + 0.1 × filename_similarity + 0.125 × weight
```

### ต้นทุนต่อเดือน (750 req/วัน)
- Infrastructure (Hetzner): ~$12
- AI (GPT OSS 120B via Groq): ~$9
- **รวม: ~$21/เดือน**

### Indexing Pipeline
1. Clone/Update documentation repo
2. Parse Markdown → split by `## headers`
3. Generate embeddings (3x per chunk: content, title, filename)
4. Generate summary ด้วย AI
5. Incremental update — เปรียบเทียบ hash ก่อน embed

## ข้อมูลที่น่าสนใจ

- ไม่ใช้ Re-ranking API เพราะเพิ่ม latency 100-200ms และค่าใช้จ่าย $0.002/query โดยไม่จำเป็น
- ใช้ Node.js Cluster ทำ multi-core processing (CPU - 1 workers)
- Indexing cron ทำทุก 12 ชั่วโมง + webhook สำหรับ trigger แบบ manual
- Checksum verification ป้องกันการปลอมแปลงประวัติสนทนา

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG (Retrieval-Augmented Generation)]]
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search: BM25 + Vector]]
- [[wiki/concepts/semantic-caching|Semantic Caching]]
- [[wiki/concepts/stateless-rag-design|Stateless RAG Design]]
