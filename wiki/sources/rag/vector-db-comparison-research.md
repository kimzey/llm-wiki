---
title: "Vector Database Comparison Research — Benchmarks & Decision Guide 2026"
type: source
source_file: raw/notes/vector-db-research-2026.md
url: ""
published: 2026-04-14
tags: [vector-database, pgvector, qdrant, pinecone, chromadb, benchmark, comparison]
related: [wiki/concepts/vector-database, wiki/concepts/pgvector, wiki/concepts/embedding]
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../../raw/notes/vector-db-research-2026.md|Original file]]
>
> Sources: [4xxi Vector DB Comparison](https://4xxi.com/articles/vector-database-comparison/) · [TigerData pgvector vs Qdrant](https://www.tigerdata.com/blog/pgvector-vs-qdrant) · [CrunchyData HNSW](https://www.crunchydata.com/blog/hnsw-indexes-with-postgres-and-pgvector) · [Shakudo Top 9 2026](https://www.shakudo.io/blog/top-9-vector-databases)

## สรุป

การเปรียบเทียบ Vector Databases ยอดนิยม 6 ตัว: pgvector, Qdrant, Pinecone, Weaviate, ChromaDB, Milvus — พร้อม benchmark ข้อมูลจริง และ decision guide เลือกให้เหมาะกับ use case

## ประเด็นสำคัญ

### Benchmark ประสิทธิภาพ (2025-2026)

| Database | Latency | QPS ที่ 99% recall (50M vectors) | เหมาะกับ |
|---------|---------|----------------------------------|---------|
| **pgvector** | 2.5ms | 471 QPS (pgvectorscale) | 0-1M vectors + PostgreSQL |
| **Qdrant** | 52ms | 41 QPS | Complex filtering, 1M+ vectors |
| **Pinecone** | 87ms | สูงมาก (managed) | Enterprise, 1B+ vectors |
| **ChromaDB** | 4.5ms | ต่ำ | Prototype/dev เท่านั้น |

### pgvector ข้อจำกัดที่ต้องรู้

- รองรับได้ดีถึง **~1M vectors** โดยไม่ต้องเพิ่ม infra
- HNSW index สำหรับ 1M rows (1536 dim): **~8GB RAM**, build time ~6 นาที
- Index ต้อง fit ใน RAM — ถ้า evict ออก performance ตก
- pgvector 0.7.0 (2025): HNSW build เร็วขึ้น 67x ด้วย binary quantization

### Decision Matrix

```
0-1M vectors + มี PostgreSQL แล้ว → pgvector (ฟรี, SQL, zero infra)
Complex metadata filtering → Qdrant (1.1x overhead คงที่ทุก filter)
Managed, 1B+ vectors, enterprise → Pinecone ($70-$300+/เดือน)
Prototype/ทดสอบ → ChromaDB (setup 0 นาที)
Open-source + Hybrid text+vector → Weaviate หรือ OpenSearch
GPU-accelerated large scale → Milvus
```

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- pgvectorscale (Timescale) แก้ปัญหา scalability ของ pgvector ดั้งเดิม — 11.4x faster กว่า Qdrant ที่ scale เดียวกัน
- "A benchmark on 10k vectors tells you nothing about 5M vectors at 1536 dims" — context สำคัญมากสำหรับการอ่าน benchmark
- Pinecone: Standard/Enterprise $70–$300+/month, ที่ 5M+ vectors อาจถึง $500-1,500/month

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

- หน้า pgvector เดิมระบุว่า "ใช้ได้ในการ production" ซึ่งถูกต้อง แต่ต้องเพิ่มบริบท: เหมาะสำหรับ <1M vectors เท่านั้น — สำหรับ scale ใหญ่กว่านั้นควรพิจารณา Qdrant หรือ dedicated vector DB

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/vector-database|Vector Database]]
- [[wiki/concepts/pgvector|pgvector]]
- [[wiki/concepts/embedding|Embedding]]
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search]]
