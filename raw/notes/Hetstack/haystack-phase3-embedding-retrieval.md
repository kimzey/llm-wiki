# 🌾 Haystack Deep Dive — Phase 3: Embedding & Vector Search

> **เป้าหมาย Phase 3:** เข้าใจ Embedding, Retriever ทุกประเภท, Hybrid Search, และการ Filter

---

## 1. Embedding คืออะไร?

**Embedding** คือการแปลงข้อความเป็น **Vector (ตัวเลขหลายมิติ)** ที่แสดงความหมายทางคณิตศาสตร์

```
"Python คือภาษาโปรแกรม" → [0.12, -0.34, 0.89, ..., 0.21]  (768 หรือ 1536 มิติ)
"Python is a programming language" → [0.11, -0.33, 0.87, ..., 0.22]  ← คล้ายกัน!
"แมวกินปลา" → [-0.45, 0.23, -0.12, ..., 0.67]  ← ต่างกัน!
```

ข้อความที่มีความหมายคล้ายกัน จะมี Vector ที่ใกล้กัน (cosine similarity สูง)

---

## 2. Document Embedders — สร้าง Embedding ให้ Document

### 2.1 SentenceTransformersDocumentEmbedder (Local Model)

```python
from haystack.components.embedders import SentenceTransformersDocumentEmbedder

# ดาวน์โหลดและรันบนเครื่อง (ฟรี)
embedder = SentenceTransformersDocumentEmbedder(
    model="BAAI/bge-m3",          # Model ที่ใช้
    device="cpu",                  # "cpu" | "cuda" | "mps"
    batch_size=32,                 # ประมวลผลครั้งละกี่ doc
    normalize_embeddings=True,     # Normalize vector
    meta_fields_to_embed=["title"] # รวม meta field เข้าไปใน embedding ด้วย
)

# ต้อง warm up ก่อน (โหลด model)
embedder.warm_up()

result = embedder.run(documents=docs)
embedded_docs = result["documents"]

print(f"Embedding dimension: {len(embedded_docs[0].embedding)}")
```

### 2.2 OpenAIDocumentEmbedder

```python
from haystack.components.embedders import OpenAIDocumentEmbedder
import os

os.environ["OPENAI_API_KEY"] = "sk-..."

embedder = OpenAIDocumentEmbedder(
    model="text-embedding-3-small",  # หรือ text-embedding-3-large
    dimensions=1536,                  # ลด dimension ได้ (3-large รองรับ)
    meta_fields_to_embed=["title", "category"]
)

result = embedder.run(documents=docs)
```

### 2.3 CohereDocumentEmbedder

```python
from haystack.components.embedders import CohereDocumentEmbedder

embedder = CohereDocumentEmbedder(
    model="embed-multilingual-v3.0",   # รองรับหลายภาษา
    api_key="cohere-api-key",
    input_type="search_document"       # "search_document" | "search_query"
)
```

### เปรียบเทียบ Embedding Models

| Model | Dimension | ภาษาไทย | ฟรี | Speed |
|---|---|---|---|---|
| `BAAI/bge-m3` | 1024 | ✅ ดีมาก | ✅ | กลาง |
| `text-embedding-3-small` | 1536 | ✅ ดี | ❌ | เร็ว |
| `text-embedding-3-large` | 3072 | ✅ ดีมาก | ❌ | ช้า |
| `embed-multilingual-v3.0` | 1024 | ✅ ดี | ❌ | เร็ว |
| `multilingual-e5-large` | 1024 | ✅ ดี | ✅ | ช้า |

---

## 3. Text Embedders — สร้าง Embedding ให้ Query

ต้องใช้คู่กับ Document Embedder เสมอ (model ต้องตรงกัน)

```python
from haystack.components.embedders import SentenceTransformersTextEmbedder

# สำหรับ query (ตอน retrieval)
text_embedder = SentenceTransformersTextEmbedder(
    model="BAAI/bge-m3"  # ต้องเป็น model เดียวกับ Document Embedder
)
text_embedder.warm_up()

result = text_embedder.run(text="Python ใช้ทำอะไรได้บ้าง?")
query_embedding = result["embedding"]
print(f"Query vector dimension: {len(query_embedding)}")
```

---

## 4. Retrievers — ค้นหา Document ที่เกี่ยวข้อง

### 4.1 InMemoryBM25Retriever (Keyword Search)

BM25 = Best Match 25 → ค้นหาแบบ TF-IDF ปรับปรุง (ไม่ต้อง Embedding)

```python
from haystack.components.retrievers.in_memory import InMemoryBM25Retriever
from haystack.document_stores.in_memory import InMemoryDocumentStore

document_store = InMemoryDocumentStore()

retriever = InMemoryBM25Retriever(
    document_store=document_store,
    top_k=5,           # คืนกี่ document
    scale_score=True,  # Normalize score เป็น 0-1
    filters=None       # Filter เพิ่มเติม
)

result = retriever.run(query="Python programming language")
docs = result["documents"]
for doc in docs:
    print(f"Score: {doc.score:.3f} | {doc.content[:100]}")
```

### 4.2 InMemoryEmbeddingRetriever (Semantic Search)

```python
from haystack.components.retrievers.in_memory import InMemoryEmbeddingRetriever

retriever = InMemoryEmbeddingRetriever(
    document_store=document_store,
    top_k=5,
    filters=None
)

# ต้องใส่ query_embedding (ไม่ใช่ query text)
result = retriever.run(query_embedding=query_embedding)
docs = result["documents"]
```

### 4.3 QdrantEmbeddingRetriever (Production Vector Search)

```python
# pip install qdrant-haystack
from haystack_integrations.components.retrievers.qdrant import QdrantEmbeddingRetriever
from haystack_integrations.document_stores.qdrant import QdrantDocumentStore

document_store = QdrantDocumentStore(
    url="http://localhost:6333",
    index="my_documents",
    embedding_dim=1024,
    similarity="cosine"   # "cosine" | "dot_product" | "l2"
)

retriever = QdrantEmbeddingRetriever(
    document_store=document_store,
    top_k=5,
    score_threshold=0.7   # ตัด document ที่ score ต่ำกว่า
)
```

### 4.4 ElasticsearchBM25Retriever

```python
# pip install elasticsearch-haystack
from haystack_integrations.components.retrievers.elasticsearch import ElasticsearchBM25Retriever
from haystack_integrations.document_stores.elasticsearch import ElasticsearchDocumentStore

document_store = ElasticsearchDocumentStore(
    hosts="http://localhost:9200",
    index="my_docs"
)

retriever = ElasticsearchBM25Retriever(
    document_store=document_store,
    top_k=10
)
```

---

## 5. Metadata Filtering — กรอง Document ตาม Metadata

### 5.1 Basic Filters

```python
from haystack.document_stores.types import FilterPolicy

# Filter ใช้ได้กับ Retriever ทุกประเภท
filters = {
    "field": "meta.year",
    "operator": "==",
    "value": 2024
}

retriever = InMemoryBM25Retriever(
    document_store=document_store,
    filters=filters,
    top_k=5
)
```

### 5.2 Filter Operators

```python
# Comparison operators
{"field": "meta.year",     "operator": "==",  "value": 2024}
{"field": "meta.score",    "operator": ">=",  "value": 0.8}
{"field": "meta.category", "operator": "!=",  "value": "spam"}
{"field": "meta.date",     "operator": ">",   "value": "2024-01-01"}

# Containment
{"field": "meta.tags",   "operator": "in",     "value": ["python", "ai"]}
{"field": "meta.status", "operator": "not in", "value": ["deleted", "draft"]}

# Boolean Logic
{
    "operator": "AND",
    "conditions": [
        {"field": "meta.year",     "operator": "==", "value": 2024},
        {"field": "meta.category", "operator": "==", "value": "finance"}
    ]
}

{
    "operator": "OR",
    "conditions": [
        {"field": "meta.author", "operator": "==", "value": "Alice"},
        {"field": "meta.author", "operator": "==", "value": "Bob"}
    ]
}

# Nested AND/OR
{
    "operator": "AND",
    "conditions": [
        {"field": "meta.year", "operator": ">=", "value": 2023},
        {
            "operator": "OR",
            "conditions": [
                {"field": "meta.type", "operator": "==", "value": "report"},
                {"field": "meta.type", "operator": "==", "value": "memo"}
            ]
        }
    ]
}
```

### 5.3 Dynamic Filters (ส่ง filter ตอน run)

```python
result = retriever.run(
    query="ผลประกอบการ Q3",
    filters={
        "operator": "AND",
        "conditions": [
            {"field": "meta.quarter", "operator": "==", "value": "Q3"},
            {"field": "meta.year",    "operator": "==", "value": 2024}
        ]
    }
)
```

---

## 6. Hybrid Search — รวม BM25 + Vector Search

Hybrid Search ให้ผลดีกว่าใช้วิธีใดวิธีหนึ่ง เพราะรวม:
- **BM25**: แม่นยำกับคำเฉพาะ (ชื่อ, keyword)
- **Semantic**: เข้าใจความหมาย (sibling, synonym)

```python
from haystack import Pipeline
from haystack.components.retrievers.in_memory import (
    InMemoryBM25Retriever,
    InMemoryEmbeddingRetriever
)
from haystack.components.joiners import DocumentJoiner
from haystack.components.embedders import SentenceTransformersTextEmbedder
from haystack.document_stores.in_memory import InMemoryDocumentStore

document_store = InMemoryDocumentStore()

# Hybrid Search Pipeline
hybrid_pipeline = Pipeline()
hybrid_pipeline.add_component("text_embedder", SentenceTransformersTextEmbedder(
    model="BAAI/bge-m3"
))
hybrid_pipeline.add_component("bm25_retriever", InMemoryBM25Retriever(
    document_store=document_store,
    top_k=10
))
hybrid_pipeline.add_component("embedding_retriever", InMemoryEmbeddingRetriever(
    document_store=document_store,
    top_k=10
))
hybrid_pipeline.add_component("joiner", DocumentJoiner(
    join_mode="reciprocal_rank_fusion",  # วิธีรวม score
    top_k=5                               # ผลลัพธ์สุดท้าย
))

# เชื่อมต่อ
hybrid_pipeline.connect("text_embedder.embedding",     "embedding_retriever.query_embedding")
hybrid_pipeline.connect("bm25_retriever.documents",    "joiner.documents")
hybrid_pipeline.connect("embedding_retriever.documents","joiner.documents")

# รัน
result = hybrid_pipeline.run({
    "text_embedder":     {"text": "คำถามของผู้ใช้"},
    "bm25_retriever":    {"query": "คำถามของผู้ใช้"}
})

docs = result["joiner"]["documents"]
```

### join_mode options

| mode | คำอธิบาย | เหมาะกับ |
|---|---|---|
| `concatenate` | รวมทุก doc ไม่ซ้ำ ไม่ปรับ score | เร็ว ง่าย |
| `merge` | เฉลี่ย score ของ doc ที่ซ้ำ | Balance |
| `reciprocal_rank_fusion` | ใช้ลำดับ ranking แทน score | **แนะนำ** |

---

## 7. Ranker — จัดลำดับผลลัพธ์ใหม่

หลัง Retrieval ใช้ Ranker เพื่อ re-rank ให้แม่นยำขึ้น

```python
from haystack.components.rankers import TransformersSimilarityRanker

ranker = TransformersSimilarityRanker(
    model="cross-encoder/ms-marco-MiniLM-L-6-v2",
    top_k=3,                # เลือก top K หลัง re-rank
    score_threshold=0.0,    # ตัด doc ที่ score ต่ำกว่า
    meta_fields_to_embed=[] # embed meta เพิ่มเติม
)
ranker.warm_up()

result = ranker.run(
    query="คำถาม",
    documents=retrieved_docs
)
ranked_docs = result["documents"]
```

### Pipeline พร้อม Ranker

```python
pipeline = Pipeline()
pipeline.add_component("embedder",  SentenceTransformersTextEmbedder(model="BAAI/bge-m3"))
pipeline.add_component("retriever", InMemoryEmbeddingRetriever(document_store, top_k=20))
pipeline.add_component("ranker",    TransformersSimilarityRanker(top_k=5))

pipeline.connect("embedder.embedding",    "retriever.query_embedding")
pipeline.connect("retriever.documents",   "ranker.documents")
# ต้องส่ง query ไปให้ ranker ด้วย
pipeline.connect("embedder",              "ranker")  # pass-through query

result = pipeline.run({
    "embedder": {"text": "คำถาม"},
    "ranker": {"query": "คำถาม"}
})
```

---

## 8. ตัวอย่าง Full Semantic Search Pipeline

```python
from haystack import Pipeline
from haystack.components.embedders import (
    SentenceTransformersDocumentEmbedder,
    SentenceTransformersTextEmbedder
)
from haystack.components.retrievers.in_memory import InMemoryEmbeddingRetriever
from haystack.components.rankers import TransformersSimilarityRanker
from haystack.components.writers import DocumentWriter
from haystack.document_stores.in_memory import InMemoryDocumentStore
from haystack.dataclasses import Document

# ── Setup ──────────────────────────────────────────────
document_store = InMemoryDocumentStore()

# ── Indexing ───────────────────────────────────────────
indexing = Pipeline()
indexing.add_component("doc_embedder", SentenceTransformersDocumentEmbedder(
    model="BAAI/bge-m3"
))
indexing.add_component("writer", DocumentWriter(document_store))
indexing.connect("doc_embedder.documents", "writer.documents")

documents = [
    Document(content="Haystack เป็น framework สำหรับสร้าง AI pipeline", meta={"topic": "haystack"}),
    Document(content="Python ใช้ในงาน data science และ machine learning", meta={"topic": "python"}),
    Document(content="RAG ย่อมาจาก Retrieval-Augmented Generation", meta={"topic": "rag"}),
]

indexing.run({"doc_embedder": {"documents": documents}})

# ── Querying ───────────────────────────────────────────
querying = Pipeline()
querying.add_component("query_embedder", SentenceTransformersTextEmbedder(
    model="BAAI/bge-m3"
))
querying.add_component("retriever", InMemoryEmbeddingRetriever(
    document_store=document_store,
    top_k=10
))
querying.add_component("ranker", TransformersSimilarityRanker(top_k=3))

querying.connect("query_embedder.embedding", "retriever.query_embedding")
querying.connect("retriever.documents",      "ranker.documents")

result = querying.run({
    "query_embedder": {"text": "เทคโนโลยี AI ใหม่ๆ"},
    "ranker":         {"query": "เทคโนโลยี AI ใหม่ๆ"}
})

for doc in result["ranker"]["documents"]:
    print(f"[{doc.score:.3f}] {doc.content}")
```

---

## 9. DocumentStore Comparison

### InMemoryDocumentStore (Development)

```python
from haystack.document_stores.in_memory import InMemoryDocumentStore

store = InMemoryDocumentStore(
    bm25_tokenization_regex=r"(?u)\b\w\w+\b",  # Tokenizer
    bm25_algorithm="BM25Plus",                  # "BM25" | "BM25Plus" | "BM25L"
    bm25_parameters={"k1": 1.5, "b": 0.75},    # BM25 parameters
    embedding_similarity_function="cosine"       # "cosine" | "dot_product"
)
```

### QdrantDocumentStore (Production)

```python
# pip install qdrant-haystack
from haystack_integrations.document_stores.qdrant import QdrantDocumentStore

store = QdrantDocumentStore(
    url="http://localhost:6333",      # หรือ ":memory:" สำหรับ test
    index="my_collection",
    embedding_dim=1024,
    similarity="cosine",
    recreate_index=False,             # True = ลบและสร้างใหม่
    return_embedding=False,           # ส่ง embedding กลับมาด้วย
    wait_result_from_api=True
)
```

```bash
# รัน Qdrant ด้วย Docker
docker run -p 6333:6333 qdrant/qdrant
```

---

## 10. สรุป Phase 3

| หัวข้อ | สิ่งที่ได้เรียนรู้ |
|---|---|
| Embedding | แปลงข้อความเป็น Vector แทนความหมาย |
| Document Embedder | สร้าง embedding ตอน indexing |
| Text Embedder | สร้าง embedding จาก query ตอน search |
| BM25 Retriever | ค้นหาด้วย keyword matching |
| Embedding Retriever | ค้นหาด้วย semantic similarity |
| Hybrid Search | รวม BM25 + Semantic ด้วย RRF |
| Ranker | Re-rank ผลลัพธ์ด้วย cross-encoder |
| Metadata Filter | กรองด้วย AND/OR/operator |

---

## 📚 Phase ถัดไป

- **Phase 4:** RAG Pipeline ขั้นสูง — Generators, Prompt Engineering, Multi-turn
