---
title: "Hybrid Search — BM25 + Vector Search"
type: concept
tags: [search, bm25, vector-search, paradedb, full-text-search, semantic-search, pgvector]
sources: [arona/README.md, arona/ARONA_GUIDE.md, arona/RAG_COMPARISON.md, haystack-phase3-embedding-retrieval.md, haystack-phase3-query-pipeline.md]
related: [wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/semantic-caching.md, wiki/concepts/haystack-framework.md]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

Hybrid Search คือการรวม BM25 (keyword-based full-text search) กับ Vector Search (semantic similarity) ในการค้นหาเดียว โดยนำจุดแข็งของทั้งสองมาเสริมกัน ได้ผลลัพธ์ดีกว่าการใช้แต่ละอย่างเดี่ยวๆ

## อธิบาย

### BM25 (Best Match 25)

BM25 คืออัลกอริทึม full-text search ที่ใช้ใน search engines ทั่วไป (รวมถึง Elasticsearch) คำนวณ relevance score จาก:
- **Term Frequency (TF)**: คำนั้นปรากฏในเอกสารบ่อยแค่ไหน
- **Inverse Document Frequency (IDF)**: คำนั้นหายากแค่ไหนใน corpus ทั้งหมด
- **Document Length Normalization**: ปรับเทียบตามความยาวเอกสาร

**จุดแข็ง**: แม่นยำมากเมื่อ query และ document ใช้คำเหมือนกันพอดี
**จุดอ่อน**: ไม่เข้าใจความหมาย ("endpoint" กับ "route" ต่างกันในสายตา BM25)

### Vector Search (Semantic Search)

แปลงทั้ง query และ document เป็น vector (high-dimensional numbers) แล้วค้นหาด้วย cosine similarity หรือ dot product

**จุดแข็ง**: เข้าใจความหมาย paraphrase ได้ — "สร้าง endpoint" ≈ "สร้าง route"
**จุดอ่อน**: อาจ miss ผลลัพธ์ที่ใช้คำเหมือนกันตรงๆ, ใช้ compute มากกว่า

### Hybrid Scoring

รวมคะแนนทั้งสองด้วย weighted formula:

**Arona formula:**
```typescript
// BM25 score (normalize แล้ว) + document weight
const bm25_score = 0.775 * r_norm + 0.275 * weight

// Vector scoring (ใน SQL)
score = 0.1 * title_similarity
      + 0.675 * content_similarity
      + 0.1 * filename_similarity
      + 0.125 * weight
```

## ประเด็นสำคัญ

### Arona's Hybrid Search SQL

```sql
WITH ranked AS (
  SELECT DISTINCT ON (d.file)
    d.*,
    (0.1 * title_embedding <#> q.embedding +
     0.675 * embedding <#> q.embedding +
     0.125 * weight * -1) AS score
  FROM documents d, q
  ORDER BY d.file, score DESC
)
SELECT * FROM ranked
WHERE score >= 0.4
ORDER BY score DESC
LIMIT topK
```

### ParadeDB (PostgreSQL + BM25 + Vector)

Arona ใช้ **ParadeDB** ซึ่งเป็น PostgreSQL ที่มี:
- `pg_search` extension สำหรับ BM25
- `pgvector` extension สำหรับ vector operations
- ทำทั้งสองใน database เดียวกัน → ลด latency

เทียบกับ LangChain ที่ต้องใช้ external retriever สำหรับ BM25 + vector store แยก

### Document Weight System
เอกสารสำคัญได้คะแนนบวกเพิ่ม:
```
essential: 1.0 → blog: 0.8 → eden: 0.7
patterns: 0.5 → unknown: 0.4 → migrate/tutorial: 0.3
```

### ทำไมไม่ใช้ Re-ranking

Re-ranking API (เช่น Cohere Rerank) ทำ second-pass ranking ด้วย AI:
- เพิ่ม latency ~100-200ms
- เพิ่มค่าใช้จ่าย ~$0.002/query
- สำหรับ documentation search, hybrid scoring เพียงพอแล้ว

### Chunk Aggregation

หลังค้นหา chunks ที่อยู่ติดกัน (adjacent) จะถูก merge กันก่อนส่งให้ AI เพื่อ:
- ให้ context ที่ต่อเนื่องมากขึ้น
- ลด number of chunks ที่ AI ต้องประมวลผล

## ตัวอย่าง

```
Query: "how to create route"

BM25 finds: docs ที่มีคำว่า "route", "create"
Vector finds: docs ที่มีความหมาย "endpoint", "path", "handler"
Hybrid: รวมทั้งสองชุด, เรียงตาม combined score
```

## Haystack Implementation

Haystack implement Hybrid Search ด้วย **DocumentJoiner + Reciprocal Rank Fusion (RRF)**:
```python
pipeline.add_component("bm25_retriever",      InMemoryBM25Retriever(store, top_k=10))
pipeline.add_component("embedding_retriever", InMemoryEmbeddingRetriever(store, top_k=10))
pipeline.add_component("joiner", DocumentJoiner(
    join_mode="reciprocal_rank_fusion",  # แนะนำ: ใช้ลำดับ ranking แทน raw score
    top_k=5
))
```

**join_mode options:**
| mode | คำอธิบาย |
|---|---|
| `concatenate` | รวมทั้งหมด ไม่ปรับ score |
| `merge` | เฉลี่ย score ของ doc ที่ซ้ำ |
| `reciprocal_rank_fusion` | ใช้ลำดับ ranking — **แนะนำสุด** |

หลัง Join สามารถต่อด้วย **TransformersSimilarityRanker** (cross-encoder) เพื่อ re-rank ให้แม่นยำขึ้นอีก

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — ขั้นตอน Retrieval ของ RAG pipeline
- [[wiki/concepts/semantic-caching|Semantic Caching]] — cache ด้วย vector similarity คล้ายกัน
- [[wiki/concepts/haystack-framework|Haystack Framework]] — implementation ด้วย DocumentJoiner(RRF)

## แหล่งที่มา

- [[wiki/sources/arona-overview|Arona System Overview]]
- [[wiki/sources/arona-rag-techniques|RAG Techniques & Comparison]]

## จาก Sellsuki RAG Agent Plan v2

### ทำไม Hybrid Search ดีกว่า Vector อย่างเดียว

- Vector-only: เข้าใจ paraphrase ได้ แต่ค้นคำเฉพาะทางไม่ดี
- BM25-only: แม่นสำหรับ exact keyword แต่ไม่เข้าใจ synonyms
- **Hybrid: ได้ทั้งสอง** — งานวิจัยปี 2024-2025 ยืนยันว่าดีกว่า vector อย่างเดียวอย่างมีนัยสำคัญ

### ParadeDB สำหรับ Hybrid Search

ParadeDB = PostgreSQL ที่มี pgvector + `pg_search` (BM25 จริง) built-in พร้อมใช้งาน ต่างจาก PostgreSQL ปกติที่มีแค่ `ts_vector` (ไม่ใช่ BM25 จริง)

```sql
-- BM25 index ใน ParadeDB
CALL paradedb.create_bm25_index(
    index_name => 'idx_docs_bm25',
    table_name => 'documents',
    key_field => 'id',
    text_fields => paradedb.field('content') || paradedb.field('title') || paradedb.field('summary')
);
```

### Reciprocal Rank Fusion (RRF) Implementation

```python
# alpha = 0.5 (balanced), k = 60
for rank, row in enumerate(bm25_results):
    scores[doc_id]["score"] += (1 - alpha) * (1 / (k + rank + 1))

for rank, row in enumerate(vec_results):
    scores[doc_id]["score"] += alpha * (1 / (k + rank + 1))
```

- [[wiki/sources/sellsuki-agent-plan-v2|Sellsuki RAG Agent Plan v2]]
- [[wiki/sources/arona-vs-langchain|Arona vs LangChain]]
- [[wiki/sources/haystack-phase3-retrieval|Haystack Phase 3 — Embedding & Retrieval]]
