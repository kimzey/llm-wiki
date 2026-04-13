---
title: "pgvector — Vector Search บน PostgreSQL"
type: concept
tags: [pgvector, vector-database, postgresql, embeddings, rag, vector-search]
sources: [Sellsuki RAG Agent - Complete Implementation Guide.md]
related: [wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/rag-chunking-strategies.md, wiki/concepts/agentic-rag.md]
created: 2026-04-14
updated: 2026-04-14
---

## สรุปสั้น

pgvector คือ PostgreSQL extension ที่เพิ่มความสามารถ vector search เข้าไปใน PostgreSQL ทำให้สามารถทำ semantic similarity search ได้โดยใช้ SQL ปกติ โดยไม่ต้องตั้ง vector database แยกต่างหาก

## อธิบาย

pgvector ทำให้ PostgreSQL กลายเป็น vector database ได้ทันที เหมาะสำหรับทีมที่ใช้ PostgreSQL อยู่แล้วและต้องการเพิ่ม RAG capability โดยไม่ต้องเรียนรู้ระบบใหม่

**เทียบกับ Vector DB เฉพาะทาง:**

| | pgvector | Pinecone | Weaviate | ChromaDB | Qdrant |
|--|---------|----------|---------|---------|--------|
| ราคา | ฟรี | มีค่าใช้จ่าย | ฟรี/Enterprise | ฟรี | ฟรี/Enterprise |
| Deploy | ไม่ต้องแยก | Managed | ต้องแยก | ง่ายมาก | ต้องแยก |
| Production | ✅ | ✅ | ✅ | prototype | ✅ |
| SQL integration | ✅ | ❌ | ❌ | ❌ | ❌ |

## ประเด็นสำคัญ

### การติดตั้งและ Schema

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE documents (
    id BIGSERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    embedding vector(1536),       -- 1536 มิติสำหรับ OpenAI
    source VARCHAR(500),
    category VARCHAR(100),
    department VARCHAR(100),
    doc_type VARCHAR(50),
    title VARCHAR(500),
    last_updated TIMESTAMP,
    chunk_index INTEGER,
    chunk_total INTEGER,
    metadata JSONB DEFAULT '{}'   -- flexible metadata
);
```

### Index Types

**IVFFlat (Inverted File with Flat Compression):**
- แบ่ง vectors เป็น clusters (lists)
- ตอนค้น จะค้นเฉพาะ cluster ที่ใกล้ที่สุด
- เหมาะ: data ไม่ update บ่อย, > 10,000 rows
- `lists = sqrt(จำนวน rows)` เช่น 10K rows → lists=100

```sql
CREATE INDEX ON documents 
USING ivfflat (embedding vector_cosine_ops) 
WITH (lists = 100);
```

**HNSW (Hierarchical Navigable Small World) — แนะนำ:**
- กราฟหลายชั้น เชื่อม vectors ที่ใกล้กัน
- ค้นหาจากชั้นบน (กว้าง) ลงล่าง (ละเอียด)
- แม่นยำสูง ไม่ต้อง train เพิ่ม data ได้เรื่อยๆ
- ใช้ memory มากกว่า สร้าง index นานกว่า

```sql
CREATE INDEX ON documents 
USING hnsw (embedding vector_cosine_ops) 
WITH (m = 16, ef_construction = 64);
```

### Distance Functions

| Operator | ประเภท | ใช้เมื่อ |
|----------|-------|---------|
| `<=>` | Cosine Distance | **แนะนำ** — วัดมุม ไม่สนขนาด |
| `<->` | L2 / Euclidean | สนทั้งทิศทางและขนาด |
| `<#>` | Inner Product (negative) | เร็วสุด เมื่อ vectors normalized |

```sql
-- Cosine similarity (1 - distance)
SELECT content, 1 - (embedding <=> '[0.1, 0.2, ...]'::vector) AS similarity
FROM documents
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 5;

-- Filter + Vector Search
SELECT content, title, 1 - (embedding <=> '[...]'::vector) AS similarity
FROM documents
WHERE category = 'hr'
ORDER BY embedding <=> '[...]'::vector
LIMIT 5;
```

### Metadata Filtering

pgvector ข้อดีสำคัญคือ filter metadata ได้ด้วย WHERE clause ธรรมดา:

```sql
-- Multi-condition filter
WHERE category IN ('hr', 'policy')
  AND department = 'HR'
  AND last_updated > '2024-01-01'

-- JSONB metadata query
WHERE metadata @> '{"doc_type": "faq"}'
```

Indexes ควรสร้างสำหรับ metadata columns ที่ filter บ่อย:
```sql
CREATE INDEX idx_documents_category ON documents(category);
CREATE INDEX idx_documents_metadata ON documents USING gin(metadata);
```

### Docker Setup

```yaml
services:
  db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: sellsuki_agent
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${DB_PASSWORD}
```

## ตัวอย่าง / กรณีศึกษา

**Sellsuki RAG Agent:** ใช้ pgvector เก็บ embeddings ของเอกสารบริษัท (Policy, FAQ, SOP ฯลฯ) พร้อม metadata filter สำหรับ department/category เพื่อให้ Agent ค้นหาข้อมูลที่ถูก scope ได้

```python
# Python search function
def search_similar(query, top_k=5, category=None, similarity_threshold=0.7):
    query_embedding = get_embedding(query)
    # SQL: filter + vector search + threshold
    results = [r for r in raw_results if r.similarity >= similarity_threshold]
    return results
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — pgvector เป็น vector store ที่ใช้ใน RAG pipeline
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking]] — chunks ที่ได้จาก chunking จะถูกเก็บใน pgvector
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search]] — pgvector รองรับ vector search ส่วน full-text search ใช้ PostgreSQL built-in tsvector

## แหล่งที่มา

[[wiki/sources/sellsuki-rag-complete-guide|Sellsuki RAG Agent - Complete Implementation Guide]]
