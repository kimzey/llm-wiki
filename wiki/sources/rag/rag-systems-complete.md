---
title: "RAG Systems — Retrieval-Augmented Generation ฉบับสมบูรณ์"
type: source
source_file: "raw/notes/rag/04_rag_systems.md"
tags: [rag, vector-database, chunking, embedding, hybrid-search, reranking]
related: [wiki/concepts/rag-retrieval-augmented-generation, wiki/concepts/vector-database, wiki/concepts/embedding, wiki/concepts/rag-chunking-strategies]
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../../raw/notes/rag/04_rag_systems.md|Original file]]

## สรุป

คู่มือ RAG ฉบับสมบูรณ์ครอบคลุม Pipeline 2 Phase (Indexing + Retrieval/Generation), Chunking Strategies 4 แบบ, Embedding Models เปรียบเทียบ, Vector Databases เปรียบเทียบ, Advanced Retrieval Techniques (Hybrid Search, Reranking, Query Expansion, HyDE), Context Assembly, RAG Evaluation ด้วย RAGAS, และ Production Patterns

## ประเด็นสำคัญ

- **RAG Phase 1 (Indexing)**: Load → Chunk → Embed → Store (ทำครั้งเดียว)
- **RAG Phase 2 (Query)**: Query → Embed → Search → Rerank → Assemble → Generate
- **Chunk Size Sweet Spot**: 200-500 tokens — น้อยเกิน (ขาดบริบท) มากเกิน (relevance ลด)
- **Hybrid Search**: α × vector_score + (1-α) × bm25_score — ดีที่สุดทั้งสองโลก
- **Reranking**: Vector search top-20 → Cross-encoder rerank → top-5 ให้ LLM
- **HyDE**: ให้ LLM สร้างเอกสารสมมุติก่อน แล้วค้นด้วยเอกสารนั้น
- **Query Expansion**: ขยาย 1 คำถามเป็น 4-5 variants → ค้นทุก variant → รวม + rerank
- **RAG vs Fine-tuning**: RAG ดีสำหรับข้อมูลเปลี่ยนบ่อย/private; Fine-tune ดีสำหรับ style/tone เฉพาะ

## ข้อมูล / หลักฐาน ที่น่าสนใจ

> [!note] Vector DB Comparison
> | Database | ประเภท | จุดเด่น | ใช้เมื่อ |
> |----------|--------|---------|---------|
> | Pinecone | Cloud | ง่าย, scale | Production |
> | Chroma | Self-host | ง่ายมาก | Prototype |
> | pgvector | PostgreSQL | ใช้ SQL เดิม | มี PostgreSQL แล้ว |
> | Qdrant | Cloud/Self | เร็ว, filter ดี | Performance |

```python
# RAGAS Evaluation
results = evaluate(dataset, metrics=[faithfulness, answer_relevancy, context_recall, context_precision])
# faithfulness: 0.92, answer_relevancy: 0.88, context_recall: 0.95
```

- Multi-tenant RAG: แยก namespace ต่อ tenant — `namespace=f"tenant_{tenant_id}"`
- Incremental Indexing: เช็ค `is_indexed(doc.id)` ก่อน index — ไม่ทำซ้ำ

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG — Retrieval-Augmented Generation]]
- [[wiki/concepts/vector-database|Vector Database]]
- [[wiki/concepts/embedding|Embedding]]
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]]
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search]]
- [[wiki/concepts/rag-evaluation|RAG Evaluation]]
