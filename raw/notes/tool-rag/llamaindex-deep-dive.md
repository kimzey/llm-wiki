# 🦙 LlamaIndex & RAG Deep Dive
## Use Case: Sellsuki Company Knowledge Bot

---

## 📚 สารบัญ

1. [LlamaIndex คืออะไร?](#1-llamaindex-คืออะไร)
2. [Ecosystem ทั้งหมดที่เกี่ยวข้อง](#2-ecosystem-ทั้งหมดที่เกี่ยวข้อง)
3. [RAG คืออะไร และทำงานอย่างไร?](#3-rag-คืออะไร-และทำงานอย่างไร)
4. [Architecture ของ Sellsuki Knowledge Bot](#4-architecture-ของ-sellsuki-knowledge-bot)
5. [Chunking Strategies](#5-chunking-strategies)
6. [Metadata & Filtering](#6-metadata--filtering)
7. [Database & Index (ParadeDB)](#7-database--index-paradedb)
8. [Embedding & Caching (Dragonfly)](#8-embedding--caching-dragonfly)
9. [Semantic Cache](#9-semantic-cache)
10. [Smart Update Pipeline](#10-smart-update-pipeline)
11. [Full Code Implementation](#11-full-code-implementation)
12. [Output Examples](#12-output-examples)
13. [Deployment & Integration](#13-deployment--integration)

---

## 1. LlamaIndex คืออะไร?

**LlamaIndex** (เดิมชื่อ GPT Index) คือ **Data Framework สำหรับ LLM Applications** โดยเฉพาะ  
เป้าหมายหลักคือ "เชื่อม LLM เข้ากับข้อมูลของคุณ" ได้อย่างมีประสิทธิภาพ

```
ข้อมูลดิบ (PDF, Docs, DB, API)
        ↓
   LlamaIndex
        ↓
   LLM ที่ฉลาดขึ้น + รู้เรื่องของคุณ
```

### สิ่งที่ LlamaIndex ทำให้คุณ

| Component | หน้าที่ |
|-----------|--------|
| **Data Connectors** | ดึงข้อมูลจาก PDF, Notion, Google Drive, DB ฯลฯ |
| **Data Indexes** | จัดระเบียบข้อมูลให้ Query ได้เร็ว |
| **Engines** | Query Engine, Chat Engine, Agent |
| **Observability** | ติดตามว่า RAG ทำงานยังไง ดีแค่ไหน |

---

## 2. Ecosystem ทั้งหมดที่เกี่ยวข้อง

### 🦙 LlamaIndex Core Components

```
llama-index
├── llama-index-core          # หัวใจหลัก: nodes, indexes, engines
├── llama-index-llms-*        # เชื่อม LLM (OpenAI, Anthropic, Ollama)
├── llama-index-embeddings-*  # Embedding models
├── llama-index-vector-stores-* # Vector DB integrations
├── llama-index-readers-*     # Data loaders (PDF, Notion, etc.)
└── llama-index-postprocessor-* # Reranking, filtering results
```

### 🔗 LlamaHub
Repository ของ community connectors — มี 300+ integrations เช่น:
- `llama-index-readers-file` — PDF, Word, CSV
- `llama-index-readers-notion` — Notion pages
- `llama-index-vector-stores-postgres` — PostgreSQL / pgvector
- `llama-index-readers-confluence` — Confluence wiki

### 🧠 LlamaParse
Service สำหรับ **parse เอกสารซับซ้อน** (PDF ที่มีตาราง, รูป, layout ยาก)  
ดีกว่า PyPDF2 หรือ pdfplumber มากสำหรับเอกสาร structured

```python
from llama_parse import LlamaParse

parser = LlamaParse(
    result_type="markdown",   # หรือ "text"
    language="th",            # รองรับภาษาไทย
    api_key="llx-..."
)
docs = parser.load_data("hr_policy.pdf")
# Output: Markdown ที่ preserve ตาราง, headers ได้ดีมาก
```

### 🤖 LlamaAgents / Workflows
Framework สำหรับสร้าง **multi-step AI workflows** และ **multi-agent systems**

```
User Question
    ↓
Router Agent → เลือก Tool ที่เหมาะสม
    ├── HR Policy Tool  → ค้นหาใน hr_docs index
    ├── IT/System Tool  → ค้นหาใน it_docs index
    └── Product Tool    → ค้นหาใน product_docs index
    ↓
Synthesizer → รวมคำตอบ → ตอบ User
```

### 📊 LlamaTrace / Arize Phoenix
**Observability** — ดูว่า retrieval ดึงถูกต้องไหม, latency เท่าไหร่, คำตอบถูกต้องแค่ไหน

---

## 3. RAG คืออะไร และทำงานอย่างไร?

**RAG = Retrieval-Augmented Generation**  
แทนที่จะให้ LLM "จำ" ทุกอย่าง → ให้มัน "ค้นหา" ก่อน แล้วค่อย "ตอบ"

### RAG Pipeline (ภาพรวม)

```
┌─────────────────────────────────────────────────────────┐
│                    INDEXING PIPELINE                     │
│  Documents → Load → Chunk → Embed → Store in VectorDB   │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                    QUERY PIPELINE                        │
│  Question → Embed → Search VectorDB → Top-K Chunks      │
│           → Rerank → Build Prompt → LLM → Answer        │
└─────────────────────────────────────────────────────────┘
```

### ทำไม RAG ดีกว่า Fine-tuning?

| | Fine-tuning | RAG |
|--|-------------|-----|
| อัปเดตข้อมูล | ต้อง train ใหม่ | แค่เพิ่ม/แก้ document |
| ค่าใช้จ่าย | สูงมาก (GPU) | ถูกกว่ามาก |
| อ้างอิงแหล่งที่มา | ทำยาก | ทำได้ง่าย |
| ข้อมูล confidential | เสี่ยง data leak | ข้อมูลอยู่ใน DB ของเรา |
| เวลาทำ | สัปดาห์ | วัน/ชั่วโมง |

---

## 4. Architecture ของ Sellsuki Knowledge Bot

```
┌──────────────────────────────────────────────────────────────────┐
│                     SELLSUKI KNOWLEDGE BOT                        │
│                                                                    │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────┐  │
│  │ LINE Bot │   │Slack Bot │   │ Web Chat │   │Cursor Plugin │  │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘   └──────┬───────┘  │
│       └──────────────┴──────────────┴─────────────────┘          │
│                              ↓                                     │
│                    ┌─────────────────┐                            │
│                    │   API Gateway   │  (FastAPI)                  │
│                    └────────┬────────┘                            │
│                             ↓                                     │
│              ┌──────────────────────────────┐                    │
│              │     LlamaIndex Query Engine   │                    │
│              │  ┌────────────────────────┐  │                    │
│              │  │  Semantic Cache Check  │  │ ← Dragonfly        │
│              │  │  (Dragonfly + Redis)   │  │                    │
│              │  └───────────┬────────────┘  │                    │
│              │              ↓ (cache miss)  │                    │
│              │  ┌────────────────────────┐  │                    │
│              │  │   Query Router         │  │                    │
│              │  │  HR / IT / Product /   │  │                    │
│              │  │  Dev / General         │  │                    │
│              │  └───────────┬────────────┘  │                    │
│              │              ↓               │                    │
│              │  ┌────────────────────────┐  │                    │
│              │  │  Vector Search         │  │ ← ParadeDB         │
│              │  │  + Full-text Search    │  │   (pgvector +      │
│              │  │  (Hybrid Search)       │  │    BM25)           │
│              │  └───────────┬────────────┘  │                    │
│              │              ↓               │                    │
│              │  ┌────────────────────────┐  │                    │
│              │  │  Reranker (Cohere)     │  │                    │
│              │  └───────────┬────────────┘  │                    │
│              │              ↓               │                    │
│              │  ┌────────────────────────┐  │                    │
│              │  │  Claude / GPT-4o       │  │                    │
│              │  │  (Answer Generation)   │  │                    │
│              │  └───────────┬────────────┘  │                    │
│              └──────────────┼───────────────┘                    │
│                             ↓                                     │
│                      ┌─────────────┐                             │
│                      │   Response  │                             │
│                      │ + Sources   │                             │
│                      │ + Metadata  │                             │
│                      └─────────────┘                             │
└──────────────────────────────────────────────────────────────────┘

DOCUMENT STORE:
┌──────────────────────────────────────────────────────┐
│  hr_policy.pdf  │  it_guide.pdf  │  product_docs/    │
│  wifi_info.pdf  │  onboarding/   │  dev_standards/   │
│       ↓                                               │
│  LlamaParse → Chunks + Metadata → Embed → ParadeDB   │
└──────────────────────────────────────────────────────┘
```

---

## 5. Chunking Strategies

Chunking คือการ **ตัดเอกสารเป็นชิ้นเล็กๆ** ก่อน embed  
วิธีที่เลือกส่งผลโดยตรงต่อคุณภาพของ RAG

### Strategy 1: Fixed-Size Chunking (baseline)

```python
from llama_index.core.node_parser import SentenceSplitter

splitter = SentenceSplitter(
    chunk_size=512,       # tokens ต่อ chunk
    chunk_overlap=50,     # overlap เพื่อไม่ให้ context ขาด
)
nodes = splitter.get_nodes_from_documents(documents)
```

**เหมาะกับ:** เอกสารทั่วไป, prose  
**ข้อเสีย:** อาจตัดกลางย่อหน้าสำคัญ

---

### Strategy 2: Semantic Chunking (แนะนำ ⭐)

```python
from llama_index.core.node_parser import SemanticSplitterNodeParser
from llama_index.embeddings.openai import OpenAIEmbedding

embed_model = OpenAIEmbedding()

splitter = SemanticSplitterNodeParser(
    buffer_size=1,              # ดูประโยคข้างเคียง
    breakpoint_percentile_threshold=95,  # threshold ตัด chunk
    embed_model=embed_model
)
nodes = splitter.get_nodes_from_documents(documents)
```

**หลักการ:** ตัด chunk เมื่อ **ความหมายเปลี่ยน** ไม่ใช่แค่ตามจำนวน token  
**เหมาะกับ:** HR Policy, Technical Docs ที่แต่ละหัวข้อมีความหมายชัดเจน

---

### Strategy 3: Hierarchical Chunking (สำหรับ Sellsuki ⭐⭐)

```python
from llama_index.core.node_parser import HierarchicalNodeParser
from llama_index.core.node_parser import get_leaf_nodes

# สร้าง 3 ระดับ: Document → Section → Sentence
parser = HierarchicalNodeParser.from_defaults(
    chunk_sizes=[2048, 512, 128]
)
#  chunk_size=2048 → Parent node (บริบทกว้าง)
#  chunk_size=512  → Child node (ค้นหา)
#  chunk_size=128  → Leaf node (precise match)

all_nodes = parser.get_nodes_from_documents(documents)
leaf_nodes = get_leaf_nodes(all_nodes)
```

ใช้คู่กับ **AutoMergingRetriever**:
```python
from llama_index.core.retrievers import AutoMergingRetriever
from llama_index.core.storage.docstore import SimpleDocumentStore

docstore = SimpleDocumentStore()
docstore.add_documents(all_nodes)

base_retriever = index.as_retriever(similarity_top_k=6)
retriever = AutoMergingRetriever(
    base_retriever,
    storage_context,
    verbose=True
)
# ถ้าดึง leaf nodes มาหลายอัน → auto merge เป็น parent
# ได้ context ที่ครบกว่า โดยไม่ต้อง send tokens เยอะ
```

---

### Strategy 4: Sentence Window (สำหรับ Q&A แบบ precise)

```python
from llama_index.core.node_parser import SentenceWindowNodeParser

parser = SentenceWindowNodeParser.from_defaults(
    window_size=3,              # เก็บ 3 ประโยครอบข้างไว้ใน metadata
    window_metadata_key="window",
    original_text_metadata_key="original_text",
)
nodes = parser.get_nodes_from_documents(documents)

# Postprocessor: ขยาย context จาก window metadata
from llama_index.core.postprocessor import MetadataReplacementPostProcessor
postprocessor = MetadataReplacementPostProcessor(
    target_metadata_key="window"
)
```

**เหมาะกับ:** คำถามที่ต้องการ context รอบข้าง เช่น "ถ้าเจ็บป่วยต้องทำยังไง?"

---

### Chunking Comparison สำหรับ Sellsuki

| เอกสาร | Strategy แนะนำ | เหตุผล |
|--------|---------------|--------|
| HR Policy (PDF) | Hierarchical | มีหมวดหมู่ชัดเจน, คำถามมักถามเรื่องเดียว |
| IT Guide | Sentence Window | ขั้นตอนต้องการ context รอบข้าง |
| Product Docs | Semantic | แต่ละ feature มีความหมายอิสระ |
| Dev Standards | Fixed-Size + overlap | Code snippets ต้องการ consistency |
| Onboarding | Hierarchical | มี sections ชัดเจน |

---

## 6. Metadata & Filtering

Metadata คือ **ข้อมูลเพิ่มเติมที่แนบกับแต่ละ chunk** ใช้สำหรับ filter, search, จำกัดสิทธิ์

### Schema Metadata สำหรับ Sellsuki

```python
from llama_index.core.schema import TextNode
from datetime import datetime

node = TextNode(
    text="พนักงานมีสิทธิ์ลาพักร้อน 10 วันต่อปี...",
    metadata={
        # === Document Info ===
        "doc_id": "hr-policy-2024-v3",
        "file_name": "hr_policy_2024.pdf",
        "file_type": "pdf",
        "page_number": 12,
        "section": "Leave Policy",
        "subsection": "Annual Leave",
        
        # === Access Control ===
        "department": ["all"],          # ["hr", "engineering", "all"]
        "access_level": "public",       # "public" | "internal" | "confidential"
        "employee_type": ["full_time", "part_time", "contract"],
        
        # === Content Classification ===
        "category": "hr",               # "hr" | "it" | "product" | "dev" | "general"
        "tags": ["leave", "annual", "vacation", "วันลา"],
        "language": "th",
        
        # === Versioning ===
        "version": "3.0",
        "effective_date": "2024-01-01",
        "last_updated": datetime.now().isoformat(),
        "is_latest": True,
        
        # === For Smart Update ===
        "content_hash": "sha256:abc123...",  # ใช้เช็คว่าเนื้อหาเปลี่ยนไหม
    }
)
```

### Metadata Filtering ใน Query

```python
from llama_index.core.vector_stores.types import (
    MetadataFilter, MetadataFilters, FilterOperator, FilterCondition
)

# กรองตาม department + category
filters = MetadataFilters(
    filters=[
        MetadataFilter(key="category", value="hr"),
        MetadataFilter(key="access_level", value="public"),
        MetadataFilter(
            key="department",
            value="engineering",
            operator=FilterOperator.CONTAINS  # array contains
        ),
    ],
    condition=FilterCondition.AND
)

query_engine = index.as_query_engine(
    filters=filters,
    similarity_top_k=5
)
response = query_engine.query("ลาพักร้อนกี่วัน?")
```

### Auto Metadata Extraction

```python
from llama_index.core.extractors import (
    TitleExtractor,
    QuestionsAnsweredExtractor,
    SummaryExtractor,
    KeywordExtractor,
)
from llama_index.core.ingestion import IngestionPipeline

pipeline = IngestionPipeline(
    transformations=[
        SentenceWindowNodeParser.from_defaults(window_size=3),
        TitleExtractor(nodes=5),           # extract ชื่อหัวข้อ
        QuestionsAnsweredExtractor(        # LLM generate คำถามที่ chunk นี้ตอบได้
            questions=3,
            llm=llm
        ),
        KeywordExtractor(keywords=5),      # extract keywords
        embed_model,                       # embed ทุก node
    ]
)

nodes = pipeline.run(documents=documents)
# แต่ละ node จะมี metadata:
# {
#   "document_title": "HR Policy 2024",
#   "questions_this_excerpt_can_answer": "ลาพักร้อนกี่วัน? ลาป่วยมีกี่วัน?...",
#   "excerpt_keywords": "ลาพักร้อน, วันลา, สิทธิ์พนักงาน..."
# }
```

---

## 7. Database & Index (ParadeDB)

**ParadeDB** = PostgreSQL + pgvector + BM25 (full-text search) ในตัวเดียว  
ทำให้ทำ **Hybrid Search** ได้โดยไม่ต้องใช้หลาย DB

### Setup ParadeDB

```bash
docker run \
  --name paradedb \
  -e POSTGRESQL_USERNAME=sellsuki \
  -e POSTGRESQL_PASSWORD=secret \
  -e POSTGRESQL_DATABASE=knowledge_db \
  -p 5432:5432 \
  -v paradedb_data:/bitnami/postgresql \
  paradedb/paradedb:latest
```

### Schema Design

```sql
-- Enable extensions
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_search;

-- Main chunks table
CREATE TABLE document_chunks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    doc_id          TEXT NOT NULL,
    file_name       TEXT,
    content         TEXT NOT NULL,
    content_hash    TEXT NOT NULL,           -- SHA256 for dedup
    embedding       vector(1536),            -- OpenAI ada-002 dimensions
    
    -- Metadata columns (for fast filtering)
    category        TEXT,
    department      TEXT[],
    access_level    TEXT DEFAULT 'public',
    tags            TEXT[],
    language        TEXT DEFAULT 'th',
    section         TEXT,
    page_number     INT,
    
    -- Versioning
    version         TEXT,
    effective_date  DATE,
    is_latest       BOOLEAN DEFAULT true,
    
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now()
);

-- BM25 index สำหรับ full-text search (ParadeDB specific)
CREATE INDEX chunks_bm25_idx ON document_chunks
USING bm25 (id, content, tags, section)
WITH (key_field='id', text_fields='{"content": {"tokenizer": {"type": "icu"}}, "section": {}}');
-- icu tokenizer รองรับภาษาไทย

-- HNSW index สำหรับ vector search (เร็วกว่า IVFFlat)
CREATE INDEX chunks_hnsw_idx ON document_chunks
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Indexes สำหรับ metadata filtering
CREATE INDEX idx_chunks_category ON document_chunks(category);
CREATE INDEX idx_chunks_access_level ON document_chunks(access_level);
CREATE INDEX idx_chunks_is_latest ON document_chunks(is_latest);
CREATE INDEX idx_chunks_content_hash ON document_chunks(content_hash);
```

### Hybrid Search Query

```sql
-- Hybrid Search: รวม Vector + BM25 ด้วย RRF (Reciprocal Rank Fusion)
WITH vector_results AS (
    SELECT
        id,
        content,
        metadata,
        1 - (embedding <=> $1::vector) AS vector_score,
        ROW_NUMBER() OVER (ORDER BY embedding <=> $1::vector) AS vector_rank
    FROM document_chunks
    WHERE is_latest = true
      AND access_level IN ('public', 'internal')
      AND ($2::text[] IS NULL OR department && $2::text[])  -- department filter
    ORDER BY embedding <=> $1::vector
    LIMIT 20
),
bm25_results AS (
    SELECT
        id,
        paradedb.score(id) AS bm25_score,
        ROW_NUMBER() OVER (ORDER BY paradedb.score(id) DESC) AS bm25_rank
    FROM document_chunks
    WHERE document_chunks @@@ paradedb.parse('content:' || $3)
      AND is_latest = true
    LIMIT 20
),
rrf_fusion AS (
    SELECT
        COALESCE(v.id, b.id) AS id,
        -- RRF formula: 1/(k + rank)
        COALESCE(1.0 / (60 + v.vector_rank), 0) +
        COALESCE(1.0 / (60 + b.bm25_rank), 0) AS rrf_score
    FROM vector_results v
    FULL OUTER JOIN bm25_results b ON v.id = b.id
)
SELECT
    dc.id,
    dc.content,
    dc.file_name,
    dc.section,
    dc.page_number,
    dc.category,
    rf.rrf_score
FROM rrf_fusion rf
JOIN document_chunks dc ON rf.id = dc.id
ORDER BY rf.rrf_score DESC
LIMIT 5;
```

### LlamaIndex + ParadeDB Integration

```python
from llama_index.vector_stores.postgres import PGVectorStore
from llama_index.core import VectorStoreIndex, StorageContext
import psycopg2

connection_string = "postgresql://sellsuki:secret@localhost:5432/knowledge_db"

vector_store = PGVectorStore.from_params(
    database="knowledge_db",
    host="localhost",
    password="secret",
    port=5432,
    user="sellsuki",
    table_name="document_chunks",
    embed_dim=1536,
    hybrid_search=True,       # เปิด Hybrid Search!
    text_search_config="thai", # Thai text search
    hnsw_kwargs={
        "hnsw_m": 16,
        "hnsw_ef_construction": 64,
        "hnsw_ef_search": 40,
        "hnsw_dist_method": "vector_cosine_ops"
    }
)

storage_context = StorageContext.from_defaults(vector_store=vector_store)
index = VectorStoreIndex(nodes, storage_context=storage_context)
```

---

## 8. Embedding & Caching (Dragonfly)

**Dragonfly** = Redis-compatible in-memory store แต่เร็วกว่า Redis มาก  
ใช้เป็น **Embedding Cache** และ **Semantic Cache**

### Setup Dragonfly

```bash
docker run \
  --name dragonfly \
  -p 6379:6379 \
  -v dragonfly_data:/data \
  docker.dragonflydb.io/dragonflydb/dragonfly:latest \
  --maxmemory=4gb \
  --save=3600 1        # snapshot ทุก 1 ชม. ถ้ามี 1 key เปลี่ยน
```

### Embedding Cache

เมื่อ Embed text เดิมซ้ำ → ดึงจาก cache แทน เซฟ cost + เวลา

```python
import redis
import hashlib
import numpy as np
import json
from typing import Optional

class EmbeddingCache:
    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.client = redis.from_url(redis_url)
        self.ttl = 60 * 60 * 24 * 30  # 30 วัน
        self.prefix = "emb:"
    
    def _make_key(self, text: str, model: str) -> str:
        # hash ของ text + model เพื่อให้ key unique
        content = f"{model}:{text}"
        return self.prefix + hashlib.sha256(content.encode()).hexdigest()
    
    def get(self, text: str, model: str) -> Optional[list[float]]:
        key = self._make_key(text, model)
        cached = self.client.get(key)
        if cached:
            return json.loads(cached)
        return None
    
    def set(self, text: str, model: str, embedding: list[float]) -> None:
        key = self._make_key(text, model)
        self.client.setex(
            key,
            self.ttl,
            json.dumps(embedding)
        )
    
    def get_or_embed(self, text: str, embed_fn, model: str = "text-embedding-3-small") -> list[float]:
        cached = self.get(text, model)
        if cached:
            return cached
        
        embedding = embed_fn(text)  # เรียก API จริง
        self.set(text, model, embedding)
        return embedding


# Custom Embedding Model ที่ใช้ Cache
from llama_index.core.embeddings import BaseEmbedding

class CachedEmbedding(BaseEmbedding):
    def __init__(self, base_embed_model, cache: EmbeddingCache):
        super().__init__()
        self._base_model = base_embed_model
        self._cache = cache
    
    def _get_text_embedding(self, text: str) -> list[float]:
        return self._cache.get_or_embed(
            text,
            lambda t: self._base_model._get_text_embedding(t),
            model=self._base_model.model_name
        )
    
    async def _aget_text_embedding(self, text: str) -> list[float]:
        return self._get_text_embedding(text)
    
    def _get_query_embedding(self, query: str) -> list[float]:
        return self._get_text_embedding(query)
    
    async def _aget_query_embedding(self, query: str) -> list[float]:
        return self._get_query_embedding(query)


# Usage
from llama_index.embeddings.openai import OpenAIEmbedding

base_embed = OpenAIEmbedding(model="text-embedding-3-small")
cache = EmbeddingCache("redis://localhost:6379")
embed_model = CachedEmbedding(base_embed, cache)
```

---

## 9. Semantic Cache

**Semantic Cache** = Cache ที่เข้าใจ "ความหมาย" ของคำถาม  
ถ้าถาม "ลาพักร้อนกี่วัน?" และเคยถาม "annual leave กี่วัน?" → ใช้คำตอบเดิมได้!

```python
import redis
import numpy as np
import json
from typing import Optional, Tuple

class SemanticCache:
    """
    Semantic Cache ที่เก็บ:
    1. Embedding ของ query
    2. คำตอบที่ได้
    
    เมื่อมี query ใหม่ → embed → cosine similarity → ถ้า > threshold → return cache
    """
    
    def __init__(
        self,
        redis_url: str = "redis://localhost:6379",
        embed_model = None,
        similarity_threshold: float = 0.92,  # 92% similar = ถือว่าเหมือนกัน
        ttl: int = 60 * 60 * 24,             # Cache 24 ชม.
        max_cache_size: int = 10000
    ):
        self.client = redis.from_url(redis_url)
        self.embed_model = embed_model
        self.threshold = similarity_threshold
        self.ttl = ttl
        self.prefix = "scache:"
        self.index_key = "scache:index"      # เก็บ list ของ cache keys ทั้งหมด
    
    def _cosine_similarity(self, a: list, b: list) -> float:
        a, b = np.array(a), np.array(b)
        return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))
    
    def get(self, query: str) -> Optional[Tuple[str, str, float]]:
        """
        Returns: (cached_answer, original_query, similarity_score) or None
        """
        query_embedding = self.embed_model._get_query_embedding(query)
        
        # ดึง keys ทั้งหมด
        cache_keys = self.client.lrange(self.index_key, 0, -1)
        
        best_match = None
        best_score = 0
        
        for key in cache_keys:
            cached_data = self.client.get(key)
            if not cached_data:
                continue
            
            data = json.loads(cached_data)
            similarity = self._cosine_similarity(
                query_embedding,
                data["embedding"]
            )
            
            if similarity > best_score:
                best_score = similarity
                best_match = data
        
        if best_score >= self.threshold and best_match:
            return (
                best_match["answer"],
                best_match["query"],
                best_score
            )
        return None
    
    def set(self, query: str, answer: str, sources: list = None) -> None:
        embedding = self.embed_model._get_query_embedding(query)
        
        import hashlib
        key = self.prefix + hashlib.md5(query.encode()).hexdigest()
        
        data = {
            "query": query,
            "answer": answer,
            "sources": sources or [],
            "embedding": embedding,
            "cached_at": __import__("datetime").datetime.now().isoformat()
        }
        
        self.client.setex(key, self.ttl, json.dumps(data))
        self.client.lpush(self.index_key, key)
        
        # ตัด index ถ้า overflow
        self.client.ltrim(self.index_key, 0, 9999)
    
    def invalidate_by_document(self, doc_id: str) -> int:
        """เมื่อ document อัปเดต → ล้าง cache ที่เกี่ยวข้อง"""
        # ใน production: เก็บ doc_id ใน cache data แล้วหา + delete
        pass


# Semantic Cache Middleware สำหรับ Query Engine
class CachedQueryEngine:
    def __init__(self, query_engine, semantic_cache: SemanticCache):
        self.engine = query_engine
        self.cache = semantic_cache
    
    def query(self, question: str, metadata: dict = None) -> dict:
        # 1. ลอง cache ก่อน
        cached = self.cache.get(question)
        if cached:
            answer, original_q, score = cached
            return {
                "answer": answer,
                "from_cache": True,
                "similarity_score": score,
                "original_cached_query": original_q,
                "sources": []
            }
        
        # 2. Cache miss → query จริง
        response = self.engine.query(question)
        
        sources = [
            {
                "file": node.metadata.get("file_name"),
                "section": node.metadata.get("section"),
                "page": node.metadata.get("page_number"),
                "score": node.score
            }
            for node in response.source_nodes
        ]
        
        # 3. บันทึก cache
        self.cache.set(question, str(response), sources)
        
        return {
            "answer": str(response),
            "from_cache": False,
            "sources": sources
        }
```

---

## 10. Smart Update Pipeline

เมื่อมีเอกสารใหม่/อัปเดต → ระบบต้องฉลาดพอที่จะ:
1. ตรวจว่า content เปลี่ยนจริงไหม (content hash)
2. ถ้าเปลี่ยน → chunk ใหม่ → embed เฉพาะ chunk ที่เปลี่ยน
3. Mark เอกสารเก่าว่า `is_latest = false`
4. Invalidate semantic cache ที่เกี่ยวข้อง

```python
import hashlib
from pathlib import Path
from typing import List, Dict
import psycopg2
from dataclasses import dataclass

@dataclass
class ChunkFingerprint:
    content_hash: str
    content: str
    metadata: dict

class SmartUpdatePipeline:
    """
    Pipeline ที่ฉลาด:
    - เช็ค hash ก่อน embed
    - ไม่ embed ซ้ำถ้าเนื้อหาไม่เปลี่ยน
    - อัปเดตเฉพาะ chunk ที่เปลี่ยนจริง
    """
    
    def __init__(self, db_conn_string: str, embed_model, embed_cache: EmbeddingCache):
        self.conn_string = db_conn_string
        self.embed_model = embed_model
        self.embed_cache = embed_cache
    
    def _hash_content(self, content: str) -> str:
        return hashlib.sha256(content.strip().encode()).hexdigest()
    
    def _get_existing_hashes(self, doc_id: str) -> Dict[str, str]:
        """ดึง content_hash ทั้งหมดของ doc_id นี้จาก DB"""
        with psycopg2.connect(self.conn_string) as conn:
            with conn.cursor() as cur:
                cur.execute(
                    "SELECT content_hash, id FROM document_chunks WHERE doc_id = %s",
                    (doc_id,)
                )
                return {row[0]: row[1] for row in cur.fetchall()}
    
    def _mark_old_as_deprecated(self, doc_id: str) -> None:
        """Mark เวอร์ชั่นเก่าว่าไม่ใช่ latest"""
        with psycopg2.connect(self.conn_string) as conn:
            with conn.cursor() as cur:
                cur.execute(
                    "UPDATE document_chunks SET is_latest = false WHERE doc_id = %s",
                    (doc_id,)
                )
                conn.commit()
    
    def process_document(self, file_path: str, doc_id: str, metadata: dict) -> dict:
        """
        Process document อัจฉริยะ:
        Returns: {"new_chunks": int, "reused_chunks": int, "skipped_chunks": int}
        """
        # 1. Load + Parse document
        from llama_parse import LlamaParse
        parser = LlamaParse(result_type="markdown", language="th")
        documents = parser.load_data(file_path)
        
        # 2. Chunk
        from llama_index.core.node_parser import HierarchicalNodeParser
        node_parser = HierarchicalNodeParser.from_defaults(chunk_sizes=[1024, 256, 64])
        nodes = node_parser.get_nodes_from_documents(documents)
        
        # 3. ดึง hash เก่าจาก DB
        existing_hashes = self._get_existing_hashes(doc_id)
        
        stats = {"new_chunks": 0, "reused_chunks": 0}
        new_nodes = []
        
        for node in nodes:
            content_hash = self._hash_content(node.text)
            node.metadata.update({**metadata, "content_hash": content_hash, "doc_id": doc_id})
            
            if content_hash in existing_hashes:
                # ✅ Content ไม่เปลี่ยน → reuse embedding เดิม
                # อัปเดต metadata เฉพาะ (เช่น is_latest = true)
                chunk_id = existing_hashes[content_hash]
                with psycopg2.connect(self.conn_string) as conn:
                    with conn.cursor() as cur:
                        cur.execute(
                            "UPDATE document_chunks SET is_latest = true, updated_at = now() WHERE id = %s",
                            (chunk_id,)
                        )
                        conn.commit()
                stats["reused_chunks"] += 1
            else:
                # ❌ Content ใหม่/เปลี่ยน → ต้อง embed ใหม่
                # ลอง embedding cache ก่อน (กรณี text เดิมเคย embed ในเอกสารอื่น)
                cached_embedding = self.embed_cache.get(node.text, "text-embedding-3-small")
                if cached_embedding:
                    node.embedding = cached_embedding
                else:
                    # Embed จริง
                    node.embedding = self.embed_model._get_text_embedding(node.text)
                    self.embed_cache.set(node.text, "text-embedding-3-small", node.embedding)
                
                new_nodes.append(node)
                stats["new_chunks"] += 1
        
        # 4. Mark เก่าเป็น deprecated
        self._mark_old_as_deprecated(doc_id)
        
        # 5. Insert nodes ใหม่เข้า DB
        if new_nodes:
            self._insert_nodes(new_nodes)
        
        print(f"✅ {doc_id}: {stats['new_chunks']} new, {stats['reused_chunks']} reused")
        return stats
    
    def _insert_nodes(self, nodes: list) -> None:
        """Bulk insert nodes พร้อม embedding เข้า ParadeDB"""
        with psycopg2.connect(self.conn_string) as conn:
            with conn.cursor() as cur:
                for node in nodes:
                    cur.execute("""
                        INSERT INTO document_chunks 
                        (content, embedding, content_hash, doc_id, file_name, 
                         category, section, page_number, department, access_level, 
                         tags, is_latest, version, effective_date)
                        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, true, %s, %s)
                    """, (
                        node.text,
                        node.embedding,
                        node.metadata.get("content_hash"),
                        node.metadata.get("doc_id"),
                        node.metadata.get("file_name"),
                        node.metadata.get("category"),
                        node.metadata.get("section"),
                        node.metadata.get("page_number"),
                        node.metadata.get("department", ["all"]),
                        node.metadata.get("access_level", "public"),
                        node.metadata.get("tags", []),
                        node.metadata.get("version"),
                        node.metadata.get("effective_date"),
                    ))
                conn.commit()
```

---

## 11. Full Code Implementation

### Project Structure

```
sellsuki-knowledge-bot/
├── src/
│   ├── ingestion/
│   │   ├── pipeline.py          # SmartUpdatePipeline
│   │   ├── chunkers.py          # Chunking strategies
│   │   └── metadata.py          # Metadata schemas
│   ├── retrieval/
│   │   ├── hybrid_search.py     # Vector + BM25
│   │   ├── reranker.py          # Cohere reranker
│   │   └── router.py            # Query router
│   ├── cache/
│   │   ├── embedding_cache.py   # Embedding cache
│   │   └── semantic_cache.py    # Semantic cache
│   ├── api/
│   │   ├── main.py              # FastAPI app
│   │   └── schemas.py           # Request/Response models
│   └── bots/
│       ├── line_bot.py          # LINE integration
│       └── slack_bot.py         # Slack integration
├── documents/                   # เอกสารบริษัท
│   ├── hr/
│   ├── it/
│   ├── product/
│   └── dev/
├── docker-compose.yml
└── requirements.txt
```

### Main RAG Engine

```python
# src/engine.py

import os
from llama_index.core import (
    VectorStoreIndex, StorageContext, Settings
)
from llama_index.llms.anthropic import Anthropic
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.vector_stores.postgres import PGVectorStore
from llama_index.core.postprocessor import SentenceTransformerRerank
from llama_index.core.query_engine import RetrieverQueryEngine
from llama_index.core.response_synthesizers import get_response_synthesizer
from llama_index.core.retrievers import VectorIndexRetriever

class SellsukiKnowledgeEngine:
    
    SYSTEM_PROMPT = """คุณคือ AI Assistant ของบริษัท Sellsuki
คุณตอบคำถามโดยอิงจากเอกสารที่บริษัทให้มาเท่านั้น

กฎการตอบ:
1. ตอบเฉพาะจากข้อมูลในเอกสาร ไม่เดา ไม่สร้างข้อมูลขึ้นมาเอง
2. ถ้าไม่มีข้อมูลในเอกสาร ให้บอกว่า "ไม่พบข้อมูลนี้ในเอกสาร กรุณาติดต่อ HR หรือผู้รับผิดชอบโดยตรง"
3. อ้างอิงแหล่งที่มาเสมอ เช่น (ที่มา: hr_policy_2024.pdf หน้า 12)
4. ตอบเป็นภาษาไทย กระชับ ชัดเจน
5. ถ้าถามหลายเรื่อง ตอบแยกเป็นข้อ"""
    
    def __init__(self):
        # LLM
        Settings.llm = Anthropic(
            model="claude-opus-4-5",
            system_prompt=self.SYSTEM_PROMPT,
            temperature=0.0,      # ไม่ creative → ตอบตรงข้อมูล
            max_tokens=1024
        )
        
        # Embedding
        base_embed = OpenAIEmbedding(model="text-embedding-3-small")
        embed_cache = EmbeddingCache("redis://localhost:6379")
        Settings.embed_model = CachedEmbedding(base_embed, embed_cache)
        
        # Vector Store (ParadeDB)
        self.vector_store = PGVectorStore.from_params(
            database="knowledge_db",
            host="localhost",
            password=os.getenv("DB_PASSWORD"),
            port=5432,
            user="sellsuki",
            table_name="document_chunks",
            embed_dim=1536,
            hybrid_search=True,
        )
        
        storage_context = StorageContext.from_defaults(
            vector_store=self.vector_store
        )
        
        # Index
        self.index = VectorStoreIndex.from_vector_store(
            vector_store=self.vector_store
        )
        
        # Reranker
        self.reranker = SentenceTransformerRerank(
            model="cross-encoder/ms-marco-MiniLM-L-2-v2",
            top_n=3
        )
        
        # Semantic Cache
        self.semantic_cache = SemanticCache(
            redis_url="redis://localhost:6379",
            embed_model=Settings.embed_model,
            similarity_threshold=0.92
        )
        
        # Build Query Engine
        self._build_engine()
    
    def _build_engine(self, access_level: str = "public", department: list = None):
        from llama_index.core.vector_stores.types import MetadataFilters, MetadataFilter
        
        filters = MetadataFilters(filters=[
            MetadataFilter(key="is_latest", value=True),
            MetadataFilter(key="access_level", value=access_level),
        ])
        
        retriever = VectorIndexRetriever(
            index=self.index,
            similarity_top_k=10,      # ดึงมา 10 ก่อน rerank
            vector_store_query_mode="hybrid",
            alpha=0.5,                # 50% vector + 50% BM25
            filters=filters
        )
        
        synthesizer = get_response_synthesizer(
            response_mode="compact",
            use_async=True
        )
        
        base_engine = RetrieverQueryEngine(
            retriever=retriever,
            response_synthesizer=synthesizer,
            node_postprocessors=[self.reranker]
        )
        
        self.engine = CachedQueryEngine(base_engine, self.semantic_cache)
    
    def query(self, question: str, user_context: dict = None) -> dict:
        """
        user_context: {
            "employee_id": "EMP001",
            "department": "engineering",
            "role": "developer"
        }
        """
        return self.engine.query(question, metadata=user_context)
```

### FastAPI Endpoint

```python
# src/api/main.py

from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from typing import Optional
import time

app = FastAPI(title="Sellsuki Knowledge Bot API")
engine = SellsukiKnowledgeEngine()

class QueryRequest(BaseModel):
    question: str
    employee_id: Optional[str] = None
    department: Optional[str] = "all"

class QueryResponse(BaseModel):
    answer: str
    sources: list
    from_cache: bool
    response_time_ms: float
    similarity_score: Optional[float] = None  # ถ้ามาจาก cache

@app.post("/query", response_model=QueryResponse)
async def query_knowledge(request: QueryRequest):
    if not request.question.strip():
        raise HTTPException(status_code=400, detail="Question cannot be empty")
    
    start = time.time()
    
    result = engine.query(
        question=request.question,
        user_context={
            "employee_id": request.employee_id,
            "department": request.department
        }
    )
    
    elapsed = (time.time() - start) * 1000
    
    return QueryResponse(
        answer=result["answer"],
        sources=result["sources"],
        from_cache=result["from_cache"],
        response_time_ms=round(elapsed, 2),
        similarity_score=result.get("similarity_score")
    )

@app.post("/ingest")
async def ingest_document(file_path: str, doc_id: str, category: str):
    """Trigger document ingestion"""
    pipeline = SmartUpdatePipeline(...)
    stats = pipeline.process_document(file_path, doc_id, {"category": category})
    return {"status": "success", **stats}
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  paradedb:
    image: paradedb/paradedb:latest
    environment:
      POSTGRESQL_USERNAME: sellsuki
      POSTGRESQL_PASSWORD: ${DB_PASSWORD}
      POSTGRESQL_DATABASE: knowledge_db
    ports:
      - "5432:5432"
    volumes:
      - paradedb_data:/bitnami/postgresql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U sellsuki"]
      interval: 10s
      timeout: 5s
      retries: 5

  dragonfly:
    image: docker.dragonflydb.io/dragonflydb/dragonfly:latest
    ports:
      - "6379:6379"
    volumes:
      - dragonfly_data:/data
    command: >
      --maxmemory=4gb
      --save=3600 1
      --loglevel=warning
    ulimits:
      memlock: -1

  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DB_PASSWORD=${DB_PASSWORD}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
    depends_on:
      paradedb:
        condition: service_healthy
      dragonfly:
        condition: service_started
    volumes:
      - ./documents:/app/documents

volumes:
  paradedb_data:
  dragonfly_data:
```

---

## 12. Output Examples

### Example 1: คำถามทั่วไป (HR Policy)

**Input:**
```
พนักงาน: "ลาพักร้อนได้กี่วัน และต้องทำยังไง?"
```

**Output:**
```json
{
  "answer": "พนักงาน Sellsuki มีสิทธิ์ลาพักร้อนดังนี้:\n\n**จำนวนวันลา:**\n- พนักงานที่ทำงานครบ 1 ปี: 10 วัน/ปี\n- พนักงานที่ทำงานครบ 3 ปี: 15 วัน/ปี\n- พนักงานที่ทำงานครบ 5 ปีขึ้นไป: 20 วัน/ปี\n\n**ขั้นตอนการลา:**\n1. แจ้งล่วงหน้าอย่างน้อย 3 วันทำการ\n2. ยื่นคำขอในระบบ HR Portal (hr.sellsuki.com)\n3. รอการอนุมัติจากหัวหน้างาน\n4. ได้รับ email ยืนยัน\n\n⚠️ หมายเหตุ: วันลาที่ไม่ได้ใช้ไม่สามารถสะสมข้ามปีได้\n\n*(ที่มา: HR Policy 2024 หน้า 12-13, หมวด Leave Policy)*",
  
  "sources": [
    {
      "file": "hr_policy_2024.pdf",
      "section": "Leave Policy - Annual Leave",
      "page": 12,
      "score": 0.94
    },
    {
      "file": "hr_policy_2024.pdf", 
      "section": "Leave Procedure",
      "page": 13,
      "score": 0.87
    }
  ],
  
  "from_cache": false,
  "response_time_ms": 1240.5
}
```

---

### Example 2: Cache Hit (คำถามคล้ายกัน)

**Input:**
```
พนักงานใหม่: "annual leave ได้กี่วัน?"
```

**Output:**
```json
{
  "answer": "พนักงาน Sellsuki มีสิทธิ์ลาพักร้อนดังนี้:...(คำตอบเดิม)",
  "from_cache": true,
  "similarity_score": 0.96,
  "original_cached_query": "ลาพักร้อนได้กี่วัน และต้องทำยังไง?",
  "response_time_ms": 45.2     // เร็วกว่า 27x!
}
```

---

### Example 3: ไม่มีข้อมูล

**Input:**
```
"เงินเดือนผมเท่าไหร่?"
```

**Output:**
```json
{
  "answer": "ไม่พบข้อมูลนี้ในเอกสารที่มีอยู่ เนื่องจากข้อมูลเงินเดือนเป็นข้อมูลส่วนตัว กรุณาติดต่อ HR โดยตรงที่ hr@sellsuki.com หรือผ่าน HR Portal",
  "sources": [],
  "from_cache": false,
  "response_time_ms": 890.1
}
```

---

### Example 4: Dev Use Case (Business Rule)

**Input:**
```
Dev: "Order status มีอะไรบ้าง และ transition rules คืออะไร?"
```

**Output:**
```json
{
  "answer": "Order Status ใน Sellsuki Platform มีดังนี้:\n\n**States:**\n- `PENDING` → รอยืนยัน\n- `CONFIRMED` → ยืนยันแล้ว\n- `PROCESSING` → กำลังดำเนินการ\n- `SHIPPED` → จัดส่งแล้ว\n- `DELIVERED` → ส่งถึงแล้ว\n- `CANCELLED` → ยกเลิก\n- `REFUNDED` → คืนเงินแล้ว\n\n**Transition Rules:**\n```\nPENDING → CONFIRMED (เมื่อ payment ผ่าน)\nCONFIRMED → PROCESSING (manual/auto trigger)\nPROCESSING → SHIPPED (เมื่อมี tracking number)\nSHIPPED → DELIVERED (webhook จาก courier)\nAny → CANCELLED (ก่อน SHIPPED เท่านั้น)\nDELIVERED → REFUNDED (ภายใน 7 วัน)\n```\n\n⚠️ ไม่สามารถ cancel order ที่ SHIPPED แล้วได้ ต้องทำผ่าน refund flow\n\n*(ที่มา: Product Specs v2.3 หน้า 45, Order Management)*",
  
  "sources": [
    {
      "file": "product_specs_v2.3.pdf",
      "section": "Order Management - State Machine",
      "page": 45,
      "score": 0.97
    }
  ],
  "from_cache": false,
  "response_time_ms": 1560.3
}
```

---

## 13. Deployment & Integration

### LINE Bot Integration

```python
# src/bots/line_bot.py
from linebot.v3 import WebhookHandler
from linebot.v3.messaging import MessagingApi, TextMessage, ReplyMessageRequest
from linebot.v3.webhooks import MessageEvent, TextMessageContent
import re

handler = WebhookHandler(os.getenv("LINE_CHANNEL_SECRET"))
line_api = MessagingApi(ApiClient(Configuration(
    access_token=os.getenv("LINE_CHANNEL_ACCESS_TOKEN")
)))

@handler.add(MessageEvent, message=TextMessageContent)
def handle_message(event):
    question = event.message.text
    
    result = engine.query(question)
    
    # Format สำหรับ LINE
    answer = result["answer"]
    if result["sources"]:
        sources_text = "\n\n📎 แหล่งที่มา:\n" + "\n".join([
            f"• {s['file']} (หน้า {s['page']})"
            for s in result["sources"][:2]
        ])
        answer += sources_text
    
    if result["from_cache"]:
        answer += "\n\n⚡ (ตอบจาก cache)"
    
    line_api.reply_message(ReplyMessageRequest(
        reply_token=event.reply_token,
        messages=[TextMessage(text=answer)]
    ))
```

### Slack Bot Integration

```python
# src/bots/slack_bot.py
from slack_bolt import App
from slack_bolt.adapter.fastapi import SlackRequestHandler

slack_app = App(token=os.getenv("SLACK_BOT_TOKEN"))

@slack_app.event("app_mention")
def handle_mention(event, say, client):
    question = re.sub(r'<@[A-Z0-9]+>', '', event["text"]).strip()
    
    # Show typing indicator
    client.chat_postEphemeral(
        channel=event["channel"],
        user=event["user"],
        text="🤔 กำลังค้นหาข้อมูล..."
    )
    
    result = engine.query(
        question=question,
        user_context={"slack_user": event["user"]}
    )
    
    # Slack Block Kit format
    blocks = [
        {"type": "section", "text": {"type": "mrkdwn", "text": result["answer"]}},
    ]
    
    if result["sources"]:
        sources_text = "\n".join([
            f"• `{s['file']}` หน้า {s['page']} (relevance: {s['score']:.0%})"
            for s in result["sources"]
        ])
        blocks.append({
            "type": "context",
            "elements": [{"type": "mrkdwn", "text": f"📎 *แหล่งที่มา:*\n{sources_text}"}]
        })
    
    say(blocks=blocks, thread_ts=event.get("ts"))
```

---

## Summary: Tech Stack ทั้งหมด

```
┌────────────────────────────────────────────────────────┐
│                  SELLSUKI KNOWLEDGE BOT                 │
│                     TECH STACK                          │
├────────────────────┬───────────────────────────────────┤
│ Framework          │ LlamaIndex (core, agents)          │
│ LLM                │ Claude (Anthropic)                 │
│ Embeddings         │ OpenAI text-embedding-3-small      │
│ Vector DB          │ ParadeDB (pgvector + BM25)         │
│ Cache              │ Dragonfly (Embedding + Semantic)   │
│ Parser             │ LlamaParse (PDF → Markdown)        │
│ Chunking           │ Hierarchical + Semantic            │
│ Reranker           │ SentenceTransformer cross-encoder  │
│ API                │ FastAPI                            │
│ Bot Channels       │ LINE Bot + Slack Bot + Web         │
│ Observability      │ Arize Phoenix / LlamaTrace         │
├────────────────────┼───────────────────────────────────┤
│ Search Mode        │ Hybrid (Vector 50% + BM25 50%)     │
│ Cache Strategy     │ Semantic (threshold 0.92)          │
│ Update Strategy    │ Content Hash + Smart Reindex       │
│ Access Control     │ Metadata Filter per query          │
└────────────────────┴───────────────────────────────────┘
```

> **ผลลัพธ์ที่ได้:** พนักงาน Sellsuki ถามคำถามใดก็ได้ → ได้คำตอบทันที พร้อมอ้างอิงแหล่งที่มา 24/7 โดยไม่รบกวน HR ✅
