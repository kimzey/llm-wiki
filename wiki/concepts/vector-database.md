---
title: "Vector Database — ฐานข้อมูลสำหรับ Embeddings"
type: concept
tags: [vector-database, embeddings, similarity-search, rag, hnsw, ann]
sources: [wiki/sources/pgvector, wiki/sources/openrag-rag-spike-research, wiki/sources/rag-glossary-data-prep, wiki/sources/rag-complete-knowledge, wiki/sources/openrag-opensearch-vector-db, wiki/sources/vector-db-comparison-research]
related: [wiki/concepts/pgvector, wiki/concepts/embedding, wiki/concepts/rag-retrieval-augmented-generation, wiki/concepts/hybrid-search-bm25-vector]
created: 2026-04-14
updated: 2026-04-14
---

## สรุปสั้น

Vector Database คือฐานข้อมูลที่ออกแบบมาเพื่อเก็บและค้นหา **vector embeddings** โดยเฉพาะ — สามารถหา vectors ที่ "ใกล้เคียงกันทางความหมาย" ได้อย่างรวดเร็ว แม้จะมีข้อมูลหลายล้านชิ้น ถือเป็น "หัวใจ" ของระบบ RAG ทุกตัว

## อธิบาย

### ทำไมต้องมี Vector Database

ฐานข้อมูลปกติ (SQL) เก็บข้อมูลเป็น rows/columns และค้นหาด้วย exact match หรือ range query แต่สำหรับ AI สิ่งที่ต้องการคือ:

```
"หาเอกสารที่ **ความหมาย** ใกล้เคียงกับ query นี้"
```

ซึ่ง SQL ปกติทำไม่ได้ Vector Database แก้ปัญหานี้โดย:
1. เก็บข้อมูลในรูป **float array** (เช่น `[0.12, -0.45, 0.78, ...]` 1536 มิติ)
2. สร้าง index พิเศษ (HNSW, IVFFlat) ที่ค้นหา nearest neighbors ได้เร็ว
3. วัด "ความใกล้" ด้วย cosine similarity, L2, หรือ dot product

### Approximate Nearest Neighbor (ANN)

การค้นหา exact nearest neighbor ใน vector ขนาดใหญ่ช้ามาก Vector DBs ใช้ **ANN algorithms** แทน:

| Algorithm | หลักการ | ข้อดี |
|-----------|---------|-------|
| **HNSW** (Hierarchical Navigable Small World) | กราฟหลายชั้น ค้นจากบนลงล่าง | แม่นยำสูง รองรับ update ได้เรื่อยๆ |
| **IVFFlat** (Inverted File) | แบ่ง vectors เป็น clusters ค้นเฉพาะ cluster ใกล้ | เร็ว เมื่อ data ไม่เปลี่ยนบ่อย |
| **DiskANN** | HNSW variant อยู่บน disk | ใช้ RAM น้อย เหมาะ data ขนาดใหญ่มาก |

### Distance Metrics

| Metric | สูตร | เหมาะกับ |
|--------|------|---------|
| **Cosine Distance** | `1 - cos(θ)` | **แนะนำสำหรับ text** — วัดมุม ไม่สนขนาด |
| **L2 (Euclidean)** | `√Σ(a-b)²` | Image embeddings, พิกัด |
| **Dot Product** | `Σ(a×b)` | เร็วสุด เมื่อ vectors normalized แล้ว |

## ประเด็นสำคัญ

### เปรียบเทียบ Vector Databases ยอดนิยม

| | pgvector | Pinecone | Qdrant | Weaviate | ChromaDB | OpenSearch |
|-|---------|---------|--------|---------|---------|-----------|
| ราคา | ฟรี | มีค่าใช้จ่าย | ฟรี/Cloud | ฟรี/Enterprise | ฟรี | ฟรี/AWS |
| Self-host | ✅ | ❌ (Managed) | ✅ | ✅ | ✅ | ✅ |
| SQL support | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Full-text hybrid | ✅ (tsvector) | ❌ | ✅ | ✅ | ❌ | ✅ |
| Metadata filter | ✅ WHERE | ✅ | ✅ | ✅ GraphQL | ✅ | ✅ DLS |
| Production-ready | ✅ | ✅ | ✅ | ✅ | ⚠️ prototype | ✅ |

### เมื่อไหรควรใช้อะไร

```
มี PostgreSQL อยู่แล้ว → pgvector (ง่ายที่สุด)
ต้องการ hybrid BM25+Vector → OpenSearch หรือ Qdrant
ต้องการ Managed ไม่อยากดูแล Server → Pinecone
ต้องการ Semantic Metadata Filtering → Weaviate
Prototype/ทดสอบ → ChromaDB
```

## ตัวอย่าง / กรณีศึกษา

**Sellsuki RAG System:** ใช้ pgvector บน PostgreSQL เก็บ embeddings ของเอกสารบริษัท พร้อม metadata filter ตาม `category`, `department` — ทำให้ HR ถามเรื่อง Policy ได้โดยไม่เจอข้อมูล Tech

**OpenRAG Platform:** ใช้ OpenSearch ซึ่งรองรับทั้ง KNN vector search (disk_ann index) และ BM25 full-text ในระบบเดียว รองรับ Row-level Security ผ่าน Document Level Security (DLS)

**Arona (Elysia Docs RAG):** ใช้ ParadeDB (PostgreSQL + BM25) ทำ Hybrid Search ผสม pgvector สำหรับ semantic search — ได้ทั้ง keyword และ semantic ในคำสั่ง SQL เดียว

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/embedding|Embedding]] — vectors ที่เก็บใน Vector DB มาจาก embedding models
- [[wiki/concepts/pgvector|pgvector]] — implementation ของ Vector DB บน PostgreSQL
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — Vector DB เป็น "คลัง" ที่ RAG ดึงข้อมูลมา
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search]] — ผสม BM25 keyword + vector similarity search

## ข้อมูล Benchmark (2025-2026)

| Database | Latency | QPS ที่ 99% recall (50M vectors) |
|---------|---------|----------------------------------|
| pgvector | 2.5ms | 471 QPS (pgvectorscale) |
| Qdrant | 52ms | 41 QPS |
| Pinecone | 87ms | สูงมาก (managed cloud) |
| ChromaDB | 4.5ms | ต่ำ (prototype only) |

**pgvector ข้อจำกัดสำคัญ:**
- รองรับดีถึง ~1M vectors — เกินนั้นควรพิจารณา dedicated DB
- HNSW index สำหรับ 1M rows (1536 dim): ~8GB RAM, build ~6 นาที
- pgvector 0.7.0 (2025): HNSW build เร็วขึ้น 67x ด้วย binary quantization

## แหล่งที่มา

- [[wiki/sources/pgvector|pgvector — Vector Search บน PostgreSQL]] (Sellsuki RAG Guide)
- [[wiki/sources/openrag-rag-spike-research|RAG Spike Research — Vector DB & Chunking]]
- [[wiki/sources/rag-glossary-data-prep|RAG Glossary & Data Preparation]]
- [[wiki/sources/openrag-opensearch-vector-db|OpenRAG — OpenSearch Vector Database]]
- [[wiki/sources/vector-db-comparison-research|Vector Database Comparison Research 2026]]
