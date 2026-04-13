---
title: "OpenRAG — OpenSearch Vector Database & Search"
type: source
source_file: raw/notes/openrag/docs-lean/phase3-opensearch.md
url: ""
published: 2026-03-17
tags: [opensearch, vector-db, knn, hybrid-search, access-control, rag]
related: [wiki/concepts/hybrid-search-bm25-vector.md, wiki/concepts/openrag-platform.md, wiki/concepts/stateless-rag-design.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/openrag/docs-lean/phase3-opensearch.md|Phase 3 — OpenSearch]]

## สรุป
OpenRAG ใช้ OpenSearch เป็นทั้ง Vector Database, Full-text Search Engine, Semantic Search Engine, และ Access Control Store ในที่เดียว รองรับ KNN search ด้วย algorithm disk_ann และ Row-level Security ระดับ document chunk

## ประเด็นสำคัญ

### OpenSearch ใน OpenRAG ทำ 4 อย่าง
1. **Vector Database** — เก็บ embedding vectors ของ document chunks
2. **Full-text Search** — ค้นหาด้วย keyword (BM25)
3. **Semantic Search** — ค้นหาด้วยความหมาย (KNN)
4. **Access Control** — เก็บ permissions ระดับ document

### Index Schema: "documents"
```json
{
  "document_id":    "keyword",
  "filename":       "keyword",
  "page":           "integer",
  "text":           "text",
  "owner":          "keyword",
  "allowed_users":  "keyword",
  "allowed_groups": "keyword",
  "embeddings": {
    "type": "knn_vector",
    "dimension": 1536,
    "method": { "name": "hnsw", "engine": "faiss" }
  }
}
```

### Search Modes ที่รองรับ
```python
# 1. Semantic Search (KNN)
"knn": { "embeddings": { "vector": query_vector, "k": 10 } }

# 2. Keyword Search (BM25)
"match": { "text": "search terms" }

# 3. Hybrid Search (KNN + BM25) — แนะนำ
"hybrid": { "queries": [{ "knn": {...} }, { "match": {...} }] }
```

### Row-Level Security (Access Control)
```python
# ทุก document chunk มี fields:
{ "owner": "alice@co.com", "allowed_users": ["bob@co.com"], "allowed_groups": ["hr-team"] }

# User เห็นเอกสารถ้า: เป็น owner OR อยู่ใน allowed_users OR อยู่ใน allowed_groups
# Filter ทำงานอัตโนมัติผ่าน DLS (Document-Level Security)
```

### KNN Algorithm: disk_ann
- DiskANN — ออกแบบสำหรับ dataset ขนาดใหญ่ เก็บ index บน disk แทน RAM
- Parameter `ef_construction: 100` — ยิ่งสูง ยิ่ง accurate แต่ build ช้า
- Parameter `m: 16` — จำนวน connections ต่อ node
- `ef_search: 100` — ยิ่งสูง ยิ่งแม่นยำ แต่ช้ากว่า

### Embedding Models ที่รองรับ (ต้อง match กับ dimension ใน index)
```python
"text-embedding-3-small": 1536 dim   # OpenAI — Default
"text-embedding-3-large": 3072 dim   # OpenAI — แม่นกว่า
"nomic-embed-text":        768 dim   # Ollama — local
"mxbai-embed-large":      1024 dim   # Ollama — local
```

**สำคัญ:** เปลี่ยน embedding model → ต้อง re-index เอกสารใหม่ทั้งหมด

### Performance Tuning
```bash
# JVM Heap (ปรับตามข้อมูล)
OPENSEARCH_JAVA_OPTS=-Xms4g -Xmx4g  # 4GB สำหรับ production

# Password ต้องซับซ้อน
OPENSEARCH_PASSWORD=At_Least_8_Chars_Upper_Lower_Number!
```

### เปรียบเทียบกับทางเลือกอื่น
| Feature | OpenSearch | Pinecone | Weaviate | pgvector |
|---------|-----------|---------|---------|---------|
| Self-hosted | ✅ | ❌ (cloud) | ✅ | ✅ |
| Full-text search | ✅ ดีมาก | ❌ | ⚠️ | ⚠️ |
| Access control | ✅ Row-level | Namespace | ✅ | ❌ |
| Scale | ✅ Large | ✅ | ✅ | ⚠️ |
| Learning curve | สูง | ง่าย | กลาง | ง่าย |

## ข้อมูล / หลักฐาน ที่น่าสนใจ
- Indexes ใน OpenRAG: `documents` (chunks), `knowledge_filters`, `api_keys`
- OpenSearch Dashboard ที่ port 5601 ใช้ monitor, browse, และ run queries
- OIDC integration: JWT ของ user ถูกส่งไป OpenSearch → apply DLS filter อัตโนมัติ

## Concepts ที่เกี่ยวข้อง
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search BM25 + Vector]]
- [[wiki/concepts/openrag-platform|OpenRAG Platform]]
