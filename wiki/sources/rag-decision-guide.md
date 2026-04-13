---
title: "RAG Framework Decision Guide — Sellsuki"
type: source
source_file: raw/notes/tool-rag/rag-decision-guide.md
url: ""
published: 2026-03-01
tags: [rag, llamaindex, langchain, paradedb, dragonfly, sellsuki, decision]
related: [wiki/concepts/llamaindex-framework.md, wiki/concepts/langchain-framework.md, wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/hybrid-search-bm25-vector.md, wiki/concepts/semantic-caching.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/tool-rag/rag-decision-guide.md|Original file]]

## สรุป

คู่มือตัดสินใจเลือก RAG framework สำหรับระบบ Knowledge Bot ของ Sellsuki (ตอบคำถามพนักงาน 24/7 อ้างอิงเอกสารจริง) บน stack: ParadeDB + Dragonfly + Docs Sellsuki (Outline) — สรุปว่าควรใช้ **LlamaIndex**

## ประเด็นสำคัญ

- **4 ตัวเลือก**: DIY, LangChain, LlamaIndex, LlamaIndex + LangChain
- **LlamaIndex ชนะ** เพราะออกแบบมาเพื่อ RAG โดยตรง, IngestionPipeline + Cache built-in, Metadata handling ดีสุด, ParadeDB native support
- **LangChain เหมาะกว่าเมื่อ**: ต้องการ Agent ซับซ้อน + integrate หลาย tools (Slack, Calendar, Jira) หรือใช้ LangSmith monitoring
- **DIY เหมาะกว่าเมื่อ**: มีเวลา 3+ เดือน, requirement แปลกมาก, ต้องการ latency ต่ำสุด
- **Time to Production**: DIY 3-4 เดือน | LangChain 3-6 สัปดาห์ | LlamaIndex 2-4 สัปดาห์

### Chunking Strategies ที่แนะนำ

| Strategy | คำอธิบาย | เหมาะกับ |
|---------|----------|---------|
| Fixed-Size | ตัดตามขนาด token | เริ่มต้น |
| Recursive Character | แบ่งตาม paragraph → sentence → word | เอกสารทั่วไป |
| Semantic Chunker | ตัดเมื่อ embedding similarity ลด | เริ่มต้นแนะนำ |
| Hierarchical / Parent-Child | Parent (1024 tokens) + Child (128 tokens) | Advanced, ได้ทั้ง precision + context |

### ParadeDB Hybrid Search Architecture

```sql
-- HNSW index สำหรับ vector search
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- BM25 index รองรับภาษาไทย (icu tokenizer)
CREATE INDEX ON documents USING bm25 (id, content)
WITH (key_field='id', text_fields='{"content": {"tokenizer": {"type": "icu"}}}');
```

Hybrid score = 0.7 × vector_score + 0.3 × bm25_score → ดีกว่าอย่างใดอย่างหนึ่ง ~15-25%

### Dragonfly Caching Architecture (2 ชั้น)

```
Request
  ↓
[Semantic Cache] similarity > 0.95? → HIT (ประหยัด LLM call ทั้งหมด)
  ↓ MISS
[Embedding Cache] hash(text) → cached? → HIT (ประหยัด embedding cost)
  ↓ MISS
[Vector Search (ParadeDB)]
  ↓
[LLM (GPT-4o)]
  ↓
Store to Semantic Cache (TTL: 1hr)
```

### Smart Update (ไม่ Embed ซ้ำ)

- Hash content ใหม่ เปรียบกับ hash เก่าใน DB
- ถ้าเหมือนกัน → update metadata เฉย ๆ
- ถ้าต่างกัน → re-chunk + IngestionCache skip chunk ที่ไม่เปลี่ยน

### Roadmap

```
Week 1-2: Setup ParadeDB + Dragonfly + LlamaIndex ingestion pipeline
Week 3-4: Chunking experiments + Metadata + Hybrid Search + วัด Recall@5/MRR
Week 5-6: Semantic Cache + Incremental update + Rate limiting + Observability
Week 7-8: LINE/Slack Bot + Web Chat + Load test
```

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- LlamaIndex IngestionCache + Dragonfly (RedisKVStore): ถ้า chunk เหมือนเดิม → ข้าม embedding ประหยัด cost อัตโนมัติ
- `QuestionsAnsweredExtractor` ของ LlamaIndex: LLM สร้างคำถามที่ chunk นี้ตอบได้อัตโนมัติ → เพิ่ม retrieval quality
- ParadeDB ICU tokenizer รองรับภาษาไทย สำคัญสำหรับ BM25 keyword search
- LangChain abstraction ซ้อนหลายชั้น debug ยาก, package split (`langchain` → `langchain-community` → `langchain-core`) ทำให้ confusing

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

- Arona (wiki ก่อนหน้า) เลือก DIY เพราะ cost control และทีมเล็ก — guide นี้แนะนำ LlamaIndex เพราะ Sellsuki use case ต้องการ incremental update และ metadata filter ที่ซับซ้อนกว่า
- ทั้งสองไม่ขัดแย้งกัน: DIY เหมาะกับ long-term + cost-sensitive, LlamaIndex เหมาะกับ time-to-production สำคัญ

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/llamaindex-framework|LlamaIndex Framework]]
- [[wiki/concepts/langchain-framework|LangChain Framework]]
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]]
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search — BM25 + Vector]]
- [[wiki/concepts/semantic-caching|Semantic Caching]]
