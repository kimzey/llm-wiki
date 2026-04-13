# RAG System for Sellsuki — Framework Decision Guide

> **เป้าหมาย:** ระบบตอบคำถามพนักงานแบบ 24/7 อ้างอิงเอกสารจริง  
> **Stack:** ParadeDB (Vector DB) + Dragonfly (Semantic Cache) + Docs Sellsuki (Outline)

---

## 1. ภาพรวม Options ที่มี

| Option | คืออะไร | ระดับ Control | ความเร็ว Dev |
|--------|---------|---------------|-------------|
| **DIY (Custom)** | เขียนทุกอย่างเอง | สูงสุด | ช้าที่สุด |
| **LangChain** | Framework รวม tools ครบ | กลาง | เร็ว |
| **LlamaIndex** | เน้น RAG/Data pipeline โดยเฉพาะ | กลาง-สูง | เร็วมาก (RAG) |
| **LlamaIndex + LangChain** | ใช้ร่วมกัน | กลาง | ปานกลาง |

---

## 2. Deep Dive แต่ละ Option

### 2.1 DIY (Custom Pipeline)

**ทำยังไง:**
```
Document → Parser → Chunker → Embedder → ParadeDB
Query → Embed → Search ParadeDB → Rerank → LLM → Response
```

**ต้องเขียนเอง:**
- Document loader (PDF, Markdown, Notion, etc.)
- Chunking strategy (fixed, semantic, recursive)
- Embedding pipeline + cache logic
- Vector search + metadata filter
- Retrieval reranking
- Prompt construction
- Semantic cache layer

**ตัวอย่างโค้ด DIY:**
```python
import psycopg2
import hashlib
from openai import OpenAI

client = OpenAI()

def embed_with_cache(text: str, redis_client) -> list[float]:
    # Embedding cache ใน Dragonfly
    cache_key = f"emb:{hashlib.md5(text.encode()).hexdigest()}"
    cached = redis_client.get(cache_key)
    if cached:
        return json.loads(cached)
    
    emb = client.embeddings.create(
        model="text-embedding-3-small", input=text
    ).data[0].embedding
    
    redis_client.setex(cache_key, 86400, json.dumps(emb))  # cache 24hr
    return emb

def search_paradedb(query: str, metadata_filter: dict, conn) -> list[dict]:
    query_emb = embed_with_cache(query)
    
    # ParadeDB pgvector query with metadata filter
    sql = """
        SELECT content, metadata, 
               1 - (embedding <=> %s::vector) AS score
        FROM documents
        WHERE metadata @> %s
        ORDER BY embedding <=> %s::vector
        LIMIT 5
    """
    cur = conn.cursor()
    cur.execute(sql, [query_emb, json.dumps(metadata_filter), query_emb])
    return cur.fetchall()
```

**ข้อดี:**
- ✅ Control 100% — ทำ chunking แปลก ๆ ได้ตามใจ
- ✅ ไม่มี abstraction overhead
- ✅ Debug ง่าย เพราะรู้ทุก step
- ✅ ไม่ติด version/breaking change ของ framework

**ข้อเสีย:**
- ❌ ใช้เวลานานมาก (เดือน+)
- ❌ ต้องเขียน edge case เอง (PDF แบบ scan, table extraction ฯลฯ)
- ❌ ทีมเล็กจะหมดแรงก่อนถึง production

---

### 2.2 LangChain

**จุดแข็ง:** Ecosystem ใหญ่, connector เยอะ, ทำ Chain/Agent ได้ง่าย  
**จุดอ่อน:** Abstraction ลึกเกินไป, debug ยาก, version เปลี่ยนบ่อย

**Architecture กับ Stack นี้:**
```
Outline Docs → LangChain Document Loaders
             → RecursiveCharacterTextSplitter / SemanticChunker
             → OpenAIEmbeddings → PGVector (ParadeDB)

Query → ConversationalRetrievalChain
      → Semantic Cache (custom Redis/Dragonfly layer)
      → LLM → Answer + Sources
```

**ตัวอย่างโค้ด LangChain:**
```python
from langchain_community.vectorstores import PGVector
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.chains import RetrievalQAWithSourcesChain
from langchain_community.cache import RedisSemanticCache
import langchain

# Semantic Cache ใน Dragonfly (Redis-compatible)
langchain.llm_cache = RedisSemanticCache(
    redis_url="redis://dragonfly:6379",
    embedding=OpenAIEmbeddings(),
    score_threshold=0.95  # ถ้า similarity > 95% ถือว่า cache hit
)

# Setup PGVector กับ ParadeDB
CONNECTION_STRING = "postgresql://user:pass@paradedb:5432/sellsuki_rag"
vectorstore = PGVector(
    connection_string=CONNECTION_STRING,
    embedding_function=OpenAIEmbeddings(),
    collection_name="sellsuki_docs",
)

# Chunking
splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=64,
    separators=["\n\n", "\n", ".", " "]
)

# Ingest with Metadata
def ingest_document(doc_content: str, metadata: dict):
    chunks = splitter.create_documents(
        [doc_content], 
        metadatas=[metadata] * 1  # แต่ละ chunk ได้ metadata ด้วย
    )
    vectorstore.add_documents(chunks)

# RAG Chain
llm = ChatOpenAI(model="gpt-4o", temperature=0)
qa_chain = RetrievalQAWithSourcesChain.from_chain_type(
    llm=llm,
    retriever=vectorstore.as_retriever(
        search_kwargs={
            "k": 5,
            "filter": {"department": "hr"}  # metadata filter
        }
    )
)

result = qa_chain({"question": "ลาพักร้อนได้กี่วัน?"})
print(result["answer"])       # คำตอบ
print(result["sources"])      # แหล่งที่มา
```

**ข้อดี:**
- ✅ Ecosystem ใหญ่ที่สุด — มี integration กับ Notion, Slack, LINE ฯลฯ
- ✅ LangSmith (observability) ดีมาก
- ✅ เหมาะถ้าอนาคตจะทำ Agent ซับซ้อน
- ✅ Community ใหญ่ หา solution ได้ง่าย

**ข้อเสีย:**
- ❌ Abstraction ซ้อนกันหลายชั้น debug ยาก
- ❌ `langchain` → `langchain-community` → `langchain-core` split ทำให้ confusing
- ❌ ถ้า RAG เป็น core use case, มี feature น้อยกว่า LlamaIndex
- ❌ Chunking / Indexing ต้องประกอบ pieces เอง

---

### 2.3 LlamaIndex

**จุดแข็ง:** ออกแบบมาเพื่อ RAG โดยตรง, Ingestion Pipeline ครบ, metadata handling ดีมาก  
**จุดอ่อน:** Agent/Chain ecosystem เล็กกว่า LangChain

**Architecture กับ Stack นี้:**
```
Docs Sellsuki (Outline) → LlamaIndex Reader
                        → IngestionPipeline
                           ├── SentenceSplitter / SemanticSplitter
                           ├── MetadataExtractor (title, section, dept)
                           └── EmbedModel → ParadeDB (PGVectorStore)

Query → QueryEngine
      ├── HyDE / MultiQuery (optional)
      ├── VectorRetriever + MetadataFilter
      ├── Reranker (CohereRerank / ColBERT)
      └── ResponseSynthesizer → Answer + Source Nodes
```

**ตัวอย่างโค้ด LlamaIndex:**
```python
from llama_index.core import VectorStoreIndex, StorageContext
from llama_index.core.ingestion import IngestionPipeline, IngestionCache
from llama_index.core.node_parser import SentenceSplitter, SemanticSplitterNodeParser
from llama_index.core.extractors import TitleExtractor, QuestionsAnsweredExtractor
from llama_index.vector_stores.postgres import PGVectorStore
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.core.postprocessor import SentenceTransformerRerank
from llama_index.storage.kvstore.redis import RedisKVStore
import redis

# Dragonfly (Redis-compatible) สำหรับ Ingestion Cache
redis_client = redis.Redis(host="dragonfly", port=6379)
cache = IngestionCache(
    cache=RedisKVStore.from_host_and_port("dragonfly", 6379),
    collection="ingestion_cache"
)

# ParadeDB Vector Store
vector_store = PGVectorStore.from_params(
    host="paradedb",
    port=5432,
    database="sellsuki_rag",
    table_name="documents",
    embed_dim=1536,
)

# Ingestion Pipeline — หัวใจของ LlamaIndex
pipeline = IngestionPipeline(
    transformations=[
        # Step 1: Chunk
        SentenceSplitter(chunk_size=512, chunk_overlap=64),
        
        # Step 2: Extract Metadata อัตโนมัติ
        TitleExtractor(nodes=5),
        QuestionsAnsweredExtractor(questions=3),  # สร้าง Q&A จาก chunk
        
        # Step 3: Embed
        OpenAIEmbedding(model="text-embedding-3-small"),
    ],
    vector_store=vector_store,
    cache=cache,  # ← ถ้า chunk เหมือนเดิม ไม่ embed ซ้ำ ✅
)

# Ingest with Metadata
from llama_index.core import Document

docs = [
    Document(
        text="พนักงานมีสิทธิ์ลาพักร้อน 10 วันต่อปี...",
        metadata={
            "source": "hr_policy_2025.md",
            "department": "hr",
            "doc_type": "policy",
            "last_updated": "2025-01-01",
            "access_level": "all"  # สำหรับ permission control
        }
    )
]

nodes = await pipeline.arun(documents=docs)  # async สำหรับ production

# Query Engine with Reranker
index = VectorStoreIndex.from_vector_store(vector_store)
reranker = SentenceTransformerRerank(model="cross-encoder/ms-marco-MiniLM-L-6-v2", top_n=3)

query_engine = index.as_query_engine(
    similarity_top_k=10,        # ดึงมา 10 ก่อน
    node_postprocessors=[reranker],  # rerank เหลือ 3
    response_mode="tree_summarize"
)

# Query with Metadata Filter
from llama_index.core.vector_stores import MetadataFilter, MetadataFilters

filters = MetadataFilters(filters=[
    MetadataFilter(key="access_level", value="all"),
    MetadataFilter(key="department", value="hr"),
])

response = await query_engine.aquery(
    "ลาพักร้อนได้กี่วัน?",
    # filters=filters  # ใช้เมื่อต้องการ filter
)

print(response.response)           # คำตอบ
for node in response.source_nodes:
    print(node.metadata["source"]) # แหล่งที่มา
    print(node.score)              # relevance score
```

**Incremental Update (ไม่ embed ซ้ำ):**
```python
# LlamaIndex IngestionCache จัดการให้อัตโนมัติ
# ถ้า chunk content เหมือนเดิม → ข้าม embedding → ประหยัด cost

# Manual update สำหรับ document ที่เปลี่ยน
async def update_document(doc_id: str, new_content: str, metadata: dict):
    # 1. ลบ nodes เก่าที่มี doc_id นี้
    vector_store.delete(ref_doc_id=doc_id)
    
    # 2. Ingest ใหม่ — pipeline cache จะ embed เฉพาะ chunk ที่เปลี่ยน
    new_doc = Document(text=new_content, metadata=metadata, id_=doc_id)
    await pipeline.arun(documents=[new_doc])
```

**ข้อดี:**
- ✅ ออกแบบมาเพื่อ RAG โดยตรง — API สะอาด ตรงไปตรงมา
- ✅ IngestionPipeline + Cache = handle incremental update ได้สวย
- ✅ Metadata extraction อัตโนมัติดีมาก
- ✅ Query modes หลากหลาย (HyDE, SubQuestionQueryEngine, RouterQueryEngine)
- ✅ ParadeDB / PGVector integration ดีมาก
- ✅ ง่ายกว่า LangChain สำหรับ RAG use case

**ข้อเสีย:**
- ❌ Agent/Tool ecosystem เล็กกว่า LangChain
- ❌ Community เล็กกว่า LangChain (แต่กำลังโต)

---

### 2.4 LlamaIndex + LangChain ร่วมกัน

**แนวคิด:** ใช้ LlamaIndex สำหรับ Indexing/Retrieval, LangChain สำหรับ Agent/Tool  

```python
# ใช้ LlamaIndex query engine ใน LangChain
from llama_index.core.langchain_helpers.agents import IndexToolConfig, LlamaIndexTool

tool_config = IndexToolConfig(
    query_engine=query_engine,
    name="sellsuki_docs",
    description="ค้นหาข้อมูลจากเอกสารบริษัท Sellsuki เช่น HR policy, วิธีใช้ระบบ",
)

tool = LlamaIndexTool.from_tool_config(tool_config)

# ใส่ใน LangChain Agent
from langchain.agents import initialize_agent
agent = initialize_agent(
    tools=[tool, slack_tool, calendar_tool],  # รวม tools อื่น
    llm=llm,
    agent="zero-shot-react-description"
)
```

**เมื่อไหร่ควรใช้:** ถ้าต้องการ RAG ที่ดี + Agent ที่ซับซ้อน (เช่น bot ที่ดู calendar + ตอบ policy พร้อมกัน)

**ข้อเสีย:** dependency 2 framework = maintain ยากขึ้น, version conflict

---

## 3. เปรียบเทียบตรง ๆ สำหรับ Usecase นี้

| เกณฑ์ | DIY | LangChain | LlamaIndex | LlamaIndex+LC |
|-------|-----|-----------|------------|---------------|
| **Time to Production** | 3-4 เดือน | 3-6 สัปดาห์ | 2-4 สัปดาห์ | 4-7 สัปดาห์ |
| **RAG Quality** | ขึ้นกับทีม | ดี | ดีที่สุด | ดีที่สุด |
| **Incremental Update** | เขียนเอง | Manual | ✅ Built-in Cache | ✅ Built-in Cache |
| **Metadata Filter** | เขียนเอง | ✅ | ✅✅ | ✅✅ |
| **Chunking Strategies** | เขียนเอง | หลายแบบ | หลากหลายมาก | หลากหลายมาก |
| **ParadeDB Support** | ✅ psycopg2 | ✅ PGVector | ✅ PGVectorStore | ✅ |
| **Dragonfly (Cache)** | ✅ redis-py | ✅ RedisSemanticCache | ✅ RedisKVStore | ✅ |
| **Debug ง่าย** | ✅ | ❌ (abstraction ลึก) | ✅ | ❌ |
| **Observability** | เขียนเอง | LangSmith ดี | LlamaTrace | ทั้งคู่ |
| **Agent/Workflow** | เขียนเอง | ✅✅ | ✅ (LlamaIndex Workflows) | ✅✅ |
| **Docs Sellsuki Connector** | API call เอง | Custom loader | Custom reader | Custom reader |
| **Long-term Maintainability** | สูง | กลาง | สูง | ต่ำ-กลาง |

---

## 4. Chunking Strategies ที่ควรทดลอง

```
ทดลองทั้ง 4 แบบ แล้ววัด Recall / Precision

1. Fixed Size Chunking
   "พนักงาน Sellsuki มีสิทธิ์ลาพักร้อน..."
   [---- 512 tokens ----][---- 512 tokens ----]
   ❌ ตัด context กลางประโยค

2. Recursive Character Splitter  
   แบ่งตาม paragraph → sentence → word
   ✅ ดีกว่า fixed แต่ยังไม่ semantic

3. Semantic Chunker (แนะนำสำหรับเริ่ม)
   วัด embedding similarity ระหว่างประโยค
   ถ้า similarity ลด → ตัด chunk ใหม่
   ✅ chunk มีความหมายครบ ไม่ตัดกลาง concept

4. Hierarchical / Parent-Child (แนะนำ advanced)
   Parent: Section ใหญ่ (1024 tokens) → เก็บ context
   Child: Sentence เล็ก (128 tokens) → เก็บ precision
   ✅ ดึง small chunk แต่ส่ง parent ให้ LLM = ได้ทั้งความเร็ว + context
```

**Metadata ที่ควรแนบทุก Chunk:**
```json
{
  "source_url": "https://docs.sellsuki.com/hr-policy",
  "doc_title": "HR Policy 2025",
  "section": "วันลา",
  "department": "hr",
  "doc_type": "policy",
  "access_level": "all",
  "last_updated": "2025-01-15",
  "language": "th",
  "chunk_index": 3,
  "parent_doc_id": "hr-policy-2025"
}
```

---

## 5. Database & Index Design (ParadeDB)

```sql
-- ParadeDB = PostgreSQL + pgvector + BM25 full-text search
-- ได้ทั้ง Vector Search + Keyword Search ในที่เดียว

CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_doc_id TEXT,                    -- สำหรับ hierarchical chunking
    content TEXT NOT NULL,
    embedding VECTOR(1536),               -- OpenAI text-embedding-3-small
    metadata JSONB,                        -- flexible metadata
    content_hash TEXT,                    -- ใช้เช็ค duplicate ก่อน embed
    chunk_index INT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Vector Index (HNSW — เร็วกว่า IVFFlat สำหรับ production)
CREATE INDEX ON documents 
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- BM25 Full-text Index (ParadeDB feature)
CREATE INDEX ON documents 
USING bm25 (id, content)
WITH (key_field='id', text_fields='{"content": {"tokenizer": {"type": "icu"}}}');
-- icu tokenizer รองรับภาษาไทย

-- JSONB Index สำหรับ metadata filter
CREATE INDEX ON documents USING gin(metadata);

-- Hybrid Search Query (Vector + BM25)
SELECT id, content, metadata,
    (0.7 * vector_score + 0.3 * bm25_score) AS hybrid_score
FROM (
    SELECT id, content, metadata,
        1 - (embedding <=> query_embedding) AS vector_score,
        paradedb.score(id) AS bm25_score
    FROM documents
    WHERE metadata @> '{"access_level": "all"}'
    ORDER BY embedding <=> query_embedding
    LIMIT 20
) sub
ORDER BY hybrid_score DESC
LIMIT 5;
```

**ทำไมต้อง Hybrid Search:**
- **Vector:** เข้าใจ semantic — "วันหยุด" = "ลาพักร้อน" ✅
- **BM25:** Keyword exact match — "WiFi SSID Sellsuki-Corp" ✅
- **Hybrid:** ได้ทั้งคู่ ดีกว่าอย่างใดอย่างหนึ่ง ~15-25%

---

## 6. Caching Architecture (Dragonfly)

```
Request: "ลาพักร้อนได้กี่วัน?"
    │
    ▼
┌─────────────────────────────────┐
│ Semantic Cache (Dragonfly)      │
│ embed query → compare vs cache  │
│ similarity > 0.95? → HIT ✅     │  ← ประหยัด LLM call ทั้งหมด
│ similarity < 0.95? → MISS ❌    │
└─────────────────────────────────┘
    │ (MISS)
    ▼
┌─────────────────────────────────┐
│ Embedding Cache (Dragonfly)     │
│ hash(text) → cached embedding?  │
│ HIT → skip OpenAI API call ✅   │  ← ประหยัด embedding cost
│ MISS → call OpenAI → cache      │
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│ Vector Search (ParadeDB)        │
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│ LLM (GPT-4o)                    │
└─────────────────────────────────┘
    │
    ▼
Store result → Semantic Cache (TTL: 1 hour)

Redis Keys Pattern:
  sem_cache:{embedding_hash}     → {answer, sources, timestamp}
  emb_cache:{text_md5}           → {embedding_vector}
  doc_meta:{doc_id}              → {metadata}         
  rate_limit:{user_id}           → {count}
```

---

## 7. Update Strategy (ไม่ Embed ซ้ำโดยไม่จำเป็น)

```python
import hashlib

async def smart_upsert_document(doc_id: str, content: str, metadata: dict):
    """
    1. Hash content ใหม่
    2. เปรียบกับ hash เก่าใน DB
    3. ถ้าเหมือนกัน → update แค่ metadata
    4. ถ้าต่างกัน → re-chunk → pipeline (LlamaIndex cache จะ embed เฉพาะ chunk ใหม่)
    """
    new_hash = hashlib.sha256(content.encode()).hexdigest()
    
    # ดึง hash เก่า
    cur.execute(
        "SELECT content_hash FROM doc_registry WHERE doc_id = %s", [doc_id]
    )
    row = cur.fetchone()
    
    if row and row[0] == new_hash:
        # Content ไม่เปลี่ยน — update metadata เฉยๆ
        cur.execute(
            "UPDATE documents SET metadata = %s, updated_at = NOW() WHERE parent_doc_id = %s",
            [json.dumps(metadata), doc_id]
        )
        return "metadata_only_update"
    
    # Content เปลี่ยน — re-ingest
    # ลบ chunks เก่า
    vector_store.delete(ref_doc_id=doc_id)
    
    # Ingest ใหม่ — IngestionCache จะ skip chunk ที่ไม่เปลี่ยน
    new_doc = Document(text=content, metadata=metadata, id_=doc_id)
    await pipeline.arun(documents=[new_doc])
    
    # อัพเดต hash
    cur.execute(
        "INSERT INTO doc_registry (doc_id, content_hash) VALUES (%s, %s) ON CONFLICT DO UPDATE SET content_hash = EXCLUDED.content_hash",
        [doc_id, new_hash]
    )
    
    # Invalidate semantic cache ที่อาจเกี่ยวข้อง
    # (invalidate by tag หรือ TTL-based ก็พอ)
    return "full_reindex"
```

---

## 8. คำแนะนำสุดท้าย: ควรเลือกอะไร?

### ✅ แนะนำ: **LlamaIndex** (สำหรับ Usecase นี้)

**เหตุผล:**
1. **RAG คือ core use case** — LlamaIndex ออกแบบมาเพื่อสิ่งนี้โดยตรง
2. **IngestionPipeline + Cache** ตอบโจทย์ "ไม่ embed ซ้ำ" ได้ out-of-box
3. **Metadata handling ดีที่สุด** — filter by department, access_level สำคัญมาก
4. **ParadeDB PGVectorStore** มี native support
5. **Dragonfly (Redis-compatible)** ใช้ RedisKVStore ได้เลย
6. **Debug ง่าย** กว่า LangChain เพราะ abstraction ไม่ลึกเกิน
7. **Chunking experiments** — มี SentenceSplitter, SemanticSplitter, HierarchicalNodeParser ครบ

### เมื่อไหร่ควรเปลี่ยนหรือเพิ่ม LangChain:
- ต้องการ Agent ที่ integrate หลาย tools (Slack API, Calendar, Jira พร้อมกัน)
- ต้องการ LangSmith สำหรับ monitoring production
- → ตอนนั้นค่อยเพิ่ม LangChain ทีหลัง ไม่ต้องเริ่มด้วย

### เมื่อไหร่ควรทำ DIY:
- ทีมมีเวลา 3+ เดือน
- มี requirement แปลกมากที่ framework ทำไม่ได้
- ต้องการ latency ต่ำสุด ๆ ระดับ microsecond
- → ยังไม่ใช่ตอนนี้

---

## 9. Recommended Roadmap

```
Week 1-2: Foundation
├── Setup ParadeDB + Dragonfly
├── LlamaIndex ingestion pipeline
├── Connect Docs Sellsuki (Outline) reader
└── Basic query engine + test ด้วย HR policy docs

Week 3-4: Quality
├── ทดลอง chunking strategies (Fixed vs Semantic vs Hierarchical)
├── เพิ่ม Metadata + Permission filter
├── Hybrid search (Vector + BM25)
└── วัด Recall@5 / MRR

Week 5-6: Production Readiness  
├── Semantic Cache (Dragonfly)
├── Embedding Cache
├── Incremental update pipeline
├── Rate limiting + Auth
└── Observability (LlamaTrace / custom logging)

Week 7-8: Interface
├── LINE Bot integration
├── Slack Bot integration  
├── Web Chat UI
└── Load test + optimize
```

---

*เอกสารนี้จัดทำสำหรับการตัดสินใจ RAG Stack ของ Sellsuki | March 2026*
