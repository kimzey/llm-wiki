---
title: "Sellsuki RAG Agent — Plan v2 (Optimized)"
type: source
source_file: raw/notes/rag-knowledge/07-sellsuki-agent-plan-v2.md
url: ""
published: 2026-01-01
tags: [rag, sellsuki, cost-optimization, hybrid-search, paradedb, dragonfly, groq, semantic-cache, incremental-indexing]
related: [wiki/sources/sellsuki-rag-complete-guide.md, wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/hybrid-search-bm25-vector.md, wiki/concepts/semantic-caching.md, wiki/concepts/pgvector.md]
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../../raw/notes/rag/07-sellsuki-agent-plan-v2.md|Original file]]


## สรุป

Plan v2 เป็นการอัปเดต implementation plan สำหรับ Sellsuki RAG Agent โดยอ้างอิง production case study (Elysia Docs RAG) นำเสนอ 3 strategy ให้เลือกตาม budget และ scale พร้อมเทคนิค cost optimization ที่ใช้จริงได้แก่ semantic caching, hybrid search, diff-based incremental indexing, summarized content และ multi-provider LLM fallback

## ประเด็นสำคัญ

### ปัญหาของ Plan v1

- Cost สูงเกินไป ($40-70/เดือน) เพราะใช้ OpenAI ทั้งหมด ไม่มี caching
- Latency ไม่ได้ optimize: ไม่มี co-location, ไม่มี cache layer
- ไม่มี anti-abuse (rate limiting, PoW)
- Framework-heavy (LangChain overhead เยอะ สำหรับ use case ง่าย)
- Chunking ยังเบสิก: ไม่มี incremental indexing, re-index ทั้งหมดทุกครั้ง

### 3 Strategy เปรียบเทียบ

| | Strategy A: Enterprise Safe | Strategy B: Balanced (แนะนำ) | Strategy C: Cost Killer |
|--|--|--|--|
| Cost/เดือน | $50-100 | $20-35 | $12-25 |
| Latency P50 | 400-800ms | 150-400ms | 100-300ms |
| LLM | OpenAI | Groq primary + fallback | Groq flex mode |
| Embedding | OpenAI | Gemini free / fastembed | Self-host BGE-M3 |
| Vector DB | Supabase (managed) | ParadeDB self-host | ParadeDB |
| Search | Vector only | Hybrid BM25 + Vector | Hybrid BM25 + Vector |
| Cache | Upstash Redis | Dragonfly co-located | Dragonfly + multi-layer |
| Framework | LangChain / LlamaIndex | DIY | DIY ทั้งหมด |
| Hosting | Cloud Run / Railway | Hetzner VPS | Hetzner (smaller) |

**แนะนำสำหรับ Sellsuki: Strategy B** — ลด cost 60% จาก Plan v1, latency ดีขึ้น 2-3x

### LLM Provider Comparison

| Provider | TTFT (ms) | Speed (tok/s) | Cost/1M in | ภาษาไทย | Free Tier |
|----------|-----------|---------------|-----------|---------|-----------|
| GPT-4o | 300-500 | ~100 | $2.50 | ★★★★★ | ❌ |
| GPT-4o-mini | 200-400 | ~150 | $0.15 | ★★★★☆ | ❌ |
| Claude Sonnet 4.5 | 300-600 | ~90 | $3.00 | ★★★★★ | ❌ |
| Gemini 2.0 Flash | 150-300 | ~200 | $0.10 | ★★★★☆ | ✅ 1500 req/day |
| Groq Llama 3.3 70B | 50-150 | ~500 | $0.05 | ★★★★☆ | ✅ 30 req/min |
| Cerebras | 30-100 | ~800+ | $0.10 | ★★★☆☆ | ✅ |

**แนะนำ:**
- Primary: Groq Llama 3.3 70B (ถูก + เร็ว + ไทยโอเค)
- Fallback: Gemini Flash (free tier)
- Complex: GPT-4o / Claude (เฉพาะ query ยาก)

### Embedding Models Comparison

- **Groq text-embedding-004** (Google): 768 มิติ, ฟรี free tier, ภาษาไทย ★★★★☆
- **BGE-M3** (self-host): 1024 มิติ, ฟรี, multilingual ★★★★☆
- **Cohere embed-multilingual-v3**: ภาษาไทยดีที่สุด ★★★★★ แต่มีค่าใช้จ่าย
- **fastembed**: 384 มิติ ฟรี self-host เร็วมาก แต่ภาษาไทย ★★★☆☆

### Database Comparison

| DB | Vector | BM25 | Cost | หมายเหตุ |
|----|--------|------|------|---------|
| PostgreSQL + pgvector | ✅ | ts_vector (ไม่ใช่ BM25 จริง) | self-host ฟรี | ต้อง setup เอง |
| **ParadeDB** | ✅ pgvector built-in | ✅ pg_search จริง | self-host ฟรี | **แนะนำ** |
| Supabase | ✅ | FTS เบสิก | Free → $25/mo | Managed ง่าย |
| ChromaDB | ✅ | ❌ | ฟรี | prototype เท่านั้น |
| LanceDB | ✅ | ✅ | ฟรี embedded | ไม่ต้อง server |

**หมายเหตุสำคัญ:** PostgreSQL ปกติไม่มี BM25 จริงๆ — ใช้ ts_vector แทน ถ้าต้องการ BM25 ที่แม่นยำต้องใช้ ParadeDB

### Cost Optimization Techniques (เรียงตาม Impact)

1. **Semantic Caching** — ลด 40-60% LLM cost (threshold cosine > 0.95)
2. **ใช้ Groq/Cerebras แทน OpenAI** — ลด 80-90% vs GPT-4o
3. **Free Embedding Model** — ลด 100% embedding cost
4. **Summarized Content** — ลด 30-50% token count
5. **Flex Mode + Retry** — ลด 20-40% LLM cost (non-realtime)
6. **PoW + Rate Limit** — ลด abuse 80%+
7. **Tool Call แทน Stuff All Context** — ลด 20-40% tokens
8. **Diff-based Incremental Indexing** — ลด 70-90% embedding cost
9. **Embedding Cache** — ลด 30-50% embedding cost
10. **DB Query Cache** — ลด latency

### Hybrid Search Architecture

```
BM25 (exact keyword match) + Vector Search (semantic) → Reciprocal Rank Fusion
```

- `alpha = 0.5` = balanced (0 = BM25 only, 1 = vector only)
- RRF k = 60 (constant)
- DB query cache ด้วย Dragonfly (TTL 5 min)

### Request Flow (Plan v2)

```
[1] Anti-abuse (rate limit + PoW)
[2] Semantic Cache Check (cosine > 0.95 → return cached)
[3] LLM Call with tool definitions
[4] Hybrid Search (BM25 + Vector + RRF + DB cache)
[5] LLM Generate Answer (summary version)
[6] Cache & Log
```

### Diff-based Incremental Indexing

เก็บ `content_hash = SHA256(chunk_content)` ทุก chunk → ตอน re-index เทียบ hash → embed เฉพาะ chunk ที่เปลี่ยน → ประหยัด 70-90% embedding cost

### Infrastructure (Strategy B)

- Server: Hetzner CX22 ($12/เดือน) + Coolify (self-host PaaS)
- DB: ParadeDB (pgvector + BM25 ในตัว)
- Cache: Dragonfly (Redis-compatible, 25x เร็วกว่า Redis)
- Co-location: App + DB + Cache อยู่ server เดียว → ลด network hop → latency ต่ำ

### Channels ที่รองรับ

LINE Bot, Slack Bot, Web Chat (Streamlit/Chainlit), Claude Code (MCP), Cursor IDE (MCP), API (third-party)

### Plan v1 vs v2 สรุป

| ประเด็น | Plan v1 | Plan v2 (B) |
|---------|---------|-------------|
| Cost/เดือน | $40-70 | $12-25 |
| Latency P50 | 500-800ms | 150-300ms |
| Search Quality | ★★★☆☆ | ★★★★★ |
| Cache Hit Rate | 0% | 40-60% |
| Indexing | Full re-index | Diff-based |

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- Groq เร็วที่สุดในแง่ TTFT (50-150ms) เหมาะ conversational RAG
- ParadeDB = PostgreSQL ที่มี pgvector + BM25 built-in ในตัวเดียว ไม่ต้องตั้งอะไรเพิ่ม
- Dragonfly เร็วกว่า Redis 25x ในบาง workload เป็น drop-in replacement
- Co-location (App + DB อยู่ server เดียว) ลด latency ได้ 50-80%
- semantic cache threshold 0.95 เหมาะกับ FAQ-style queries

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

Plan v1 (file แรก: sellsuki-rag-complete-guide) แนะนำ LangChain สำหรับ Sellsuki — Plan v2 แนะนำ DIY แทน เหตุผล: LangChain overhead เยอะเกินสำหรับ use case ง่ายๆ, dependencies เยอะ break บ่อย ทั้งสอง approach ถูกต้องขึ้นอยู่กับ skill และ requirement

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]]
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search BM25 + Vector]]
- [[wiki/concepts/semantic-caching|Semantic Caching]]
- [[wiki/concepts/pgvector|pgvector]]
- [[wiki/concepts/agentic-rag|Agentic RAG]]
