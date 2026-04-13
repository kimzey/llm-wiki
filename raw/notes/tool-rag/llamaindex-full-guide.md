# LlamaIndex Full Guide — Sellsuki RAG System

> ครอบคลุมทุกอย่างตั้งแต่ต้องทำอะไรบ้าง → ผลลัพธ์ → Server → TypeScript → ข้อมูลที่ควรรู้

---

## 1. สิ่งที่ต้องทำทั้งหมด (Checklist)

### Phase 1: Infrastructure Setup
- [ ] ติดตั้ง ParadeDB (PostgreSQL + pgvector + BM25)
- [ ] ติดตั้ง Dragonfly (Redis-compatible, สำหรับ cache)
- [ ] สร้าง Database schema + Index
- [ ] ทดสอบ connection ทั้งสองตัว

### Phase 2: Ingestion Pipeline
- [ ] เขียน Document Reader (ดึงจาก Docs Sellsuki / Outline API)
- [ ] เลือก + ทดลอง Chunking strategy
- [ ] ออกแบบ Metadata schema
- [ ] เชื่อม Embedding model (OpenAI / local)
- [ ] เชื่อม IngestionCache กับ Dragonfly
- [ ] ทดสอบ Incremental update

### Phase 3: Query Pipeline
- [ ] สร้าง VectorStoreIndex บน ParadeDB
- [ ] ตั้งค่า Hybrid Search (Vector + BM25)
- [ ] เพิ่ม Reranker
- [ ] ตั้งค่า Metadata filter / Permission
- [ ] ทดสอบ Retrieval quality (Recall@5)

### Phase 4: Caching Layer
- [ ] Embedding Cache (Dragonfly)
- [ ] Semantic Cache (ถาม query คล้ายกัน → ตอบจาก cache)
- [ ] กำหนด TTL strategy

### Phase 5: API Server
- [ ] สร้าง REST / WebSocket API
- [ ] Authentication + Rate limiting
- [ ] Streaming response
- [ ] Source citation ในทุก response

### Phase 6: Interface
- [ ] Web Chat UI
- [ ] LINE Bot
- [ ] Slack Bot
- [ ] (Optional) Cursor/IDE MCP server

### Phase 7: Observability & Ops
- [ ] Logging (query, latency, cache hit rate)
- [ ] Tracing (LlamaTrace หรือ custom)
- [ ] Dashboard
- [ ] Alert เมื่อ quality ตก

---

## 2. สิ่งที่ได้เมื่อทำเสร็จ (Deliverables)

```
┌─────────────────────────────────────────────────────────┐
│                  Sellsuki RAG System                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Interfaces          API Layer         Data Layer       │
│  ┌──────────┐       ┌──────────┐      ┌─────────────┐  │
│  │ Web Chat │──────▶│          │      │  ParadeDB   │  │
│  ├──────────┤       │  FastAPI │      │  (Vectors + │  │
│  │ LINE Bot │──────▶│  Server  │◀────▶│  BM25 +     │  │
│  ├──────────┤       │          │      │  Metadata)  │  │
│  │Slack Bot │──────▶│          │      └─────────────┘  │
│  ├──────────┤       └────┬─────┘             │         │
│  │  Cursor  │            │            ┌─────────────┐  │
│  │  (MCP)   │            │            │  Dragonfly  │  │
│  └──────────┘            │            │  (Semantic  │  │
│                    ┌─────▼─────┐      │   Cache +   │  │
│                    │LlamaIndex │      │  Emb Cache) │  │
│                    │  Engine   │◀────▶└─────────────┘  │
│                    └───────────┘                        │
│                                                         │
│  Admin Panel                                            │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Upload Doc | Re-index | Cache Stats | Query Log  │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**สิ่งที่ได้ครบ:**
- ✅ ระบบตอบคำถามภาษาไทย/อังกฤษ 24/7
- ✅ ทุก response มี source citation (ชื่อไฟล์ + section)
- ✅ Permission-aware (พนักงานทั่วไป vs HR vs Dev เห็น doc ต่างกัน)
- ✅ อัพเดตเอกสาร → ระบบรู้อัตโนมัติภายใน N นาที
- ✅ Semantic cache ทำให้คำถามซ้ำ ๆ ตอบเร็วมาก (<100ms)
- ✅ Admin ดูได้ว่าพนักงานถามอะไร ตอบถูกไหม

---

## 3. ต้องทำ Server ไหม?

**คำตอบสั้น: ใช่ ต้องมี Server**

### ทำไมต้องมี?

```
LINE/Slack/Web → ต้องการ HTTPS endpoint รับ webhook/request
LlamaIndex     → เป็น Python library ไม่ใช่ service
                 ต้องมีตัวห่อเป็น API server
```

### Server ที่ต้องมี

```
┌─────────────────────────────────────────────────────┐
│ 1. API Server (FastAPI)       ← หลัก                │
│    - POST /query              ← รับคำถาม             │
│    - POST /ingest             ← upload/update doc    │
│    - GET  /health                                    │
│    - WebSocket /stream        ← streaming response   │
│                                                      │
│ 2. ParadeDB (PostgreSQL)      ← Vector DB            │
│    - Docker container หรือ managed                   │
│                                                      │
│ 3. Dragonfly                  ← Cache                │
│    - Docker container หรือ managed                   │
│                                                      │
│ 4. (Optional) Worker          ← Background jobs      │
│    - Celery / ARQ             ← re-index ตอน doc update│
│    - ไม่ block API server                            │
└─────────────────────────────────────────────────────┘
```

### Hosting Options

| Option | เหมาะกับ | ราคาประมาณ |
|--------|---------|-----------|
| **Docker Compose (on-prem)** | เริ่มต้น / dev | เซิร์ฟเวอร์ที่มีอยู่ |
| **Railway / Render** | MVP เร็ว | $20-50/เดือน |
| **AWS ECS / GCP Cloud Run** | Production | $50-200/เดือน |
| **Kubernetes** | Scale ใหญ่ | มี DevOps แล้ว |

### Minimal Docker Compose

```yaml
# docker-compose.yml
services:
  api:
    build: ./api
    ports: ["8000:8000"]
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - PARADEDB_URL=postgresql://user:pass@paradedb:5432/rag
      - DRAGONFLY_URL=redis://dragonfly:6379
    depends_on: [paradedb, dragonfly]

  paradedb:
    image: paradedb/paradedb:latest
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=rag
    volumes: ["paradedb_data:/var/lib/postgresql/data"]
    ports: ["5432:5432"]

  dragonfly:
    image: docker.dragonflydb.io/dragonflydb/dragonfly:latest
    ports: ["6379:6379"]
    volumes: ["dragonfly_data:/data"]

volumes:
  paradedb_data:
  dragonfly_data:
```

---

## 4. TypeScript ได้ไหม?

### คำตอบ: ได้ แต่มีข้อจำกัดสำคัญ

```
LlamaIndex มี 2 version:
  1. llamaindex (Python)     — feature ครบที่สุด ✅✅✅
  2. llamaindex-ts (TS/JS)   — feature น้อยกว่า ✅✅
```

### LlamaIndex TypeScript — ทำได้อะไรบ้าง

```typescript
// ติดตั้ง
// npm install llamaindex @llamaindex/openai @llamaindex/postgres

import { 
  Document, VectorStoreIndex, 
  Settings, OpenAI, OpenAIEmbedding 
} from "llamaindex";
import { PGVectorStore } from "@llamaindex/postgres";

// Setup
Settings.llm = new OpenAI({ model: "gpt-4o" });
Settings.embedModel = new OpenAIEmbedding({ model: "text-embedding-3-small" });

// Vector Store กับ ParadeDB
const vectorStore = new PGVectorStore({
  connectionString: process.env.PARADEDB_URL!,
  tableName: "documents",
  dimensions: 1536,
});

// Ingest Document
const doc = new Document({
  text: "พนักงาน Sellsuki มีสิทธิ์ลาพักร้อน 10 วันต่อปี",
  metadata: {
    source: "hr_policy.md",
    department: "hr",
    access_level: "all",
  },
});

const index = await VectorStoreIndex.fromDocuments([doc], { vectorStore });

// Query
const queryEngine = index.asQueryEngine({ similarityTopK: 5 });
const response = await queryEngine.query({
  query: "ลาพักร้อนได้กี่วัน?",
});

console.log(response.message.content);  // คำตอบ
console.log(response.sourceNodes);      // source
```

### เปรียบเทียบ Python vs TypeScript

| Feature | Python | TypeScript |
|---------|--------|------------|
| **IngestionPipeline** | ✅ ครบ | ⚠️ ต้องทำเอง |
| **IngestionCache** | ✅ built-in | ❌ ไม่มี ต้องเขียนเอง |
| **SemanticSplitter** | ✅ | ❌ |
| **HierarchicalNodeParser** | ✅ | ❌ |
| **QuestionsAnsweredExtractor** | ✅ | ❌ |
| **Reranker (Cohere, ColBERT)** | ✅ | ⚠️ บางตัว |
| **PGVectorStore** | ✅ | ✅ |
| **Redis Cache** | ✅ | ⚠️ ต้องทำเอง |
| **HyDE** | ✅ | ⚠️ |
| **SubQuestionQueryEngine** | ✅ | ✅ |
| **Community / examples** | ✅✅✅ | ✅✅ |
| **Streaming** | ✅ | ✅ |

### แนะนำ Architecture สำหรับทีม TypeScript

```
┌─────────────────────────────────────────────────────┐
│  Option A: Full TypeScript (ถ้าทีมไม่มี Python)     │
│                                                     │
│  Next.js / Express API (TS)                         │
│    ↓                                                │
│  LlamaIndex TS — query เท่านั้น                    │
│    ↓                                                │
│  ParadeDB (pgvector query ด้วย SQL ตรง ๆ ก็ได้)    │
│                                                     │
│  ⚠️ Ingestion pipeline ต้องเขียนเองบางส่วน          │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  Option B: Python Backend + TypeScript Frontend (แนะนำ)│
│                                                     │
│  TypeScript (Next.js / Remix)  ← Frontend / UI      │
│    ↓ HTTP                                           │
│  Python FastAPI                ← RAG Engine         │
│    ↓                                                │
│  LlamaIndex Python             ← Full feature       │
│    ↓                          ↓                     │
│  ParadeDB                    Dragonfly               │
│                                                     │
│  ✅ TS team ทำ UI/API layer                         │
│  ✅ Python team (หรือ solo) ทำ RAG core             │
└─────────────────────────────────────────────────────┘
```

---

## 5. Architecture รวมทั้งหมดแบบละเอียด

```
[Document Sources]
  Docs Sellsuki (Outline)  →  Outline API Reader
  Google Drive / Notion    →  LlamaIndex Readers
  Manual Upload (PDF/MD)   →  File Loaders
         │
         ▼
[Ingestion Pipeline]  (Python — LlamaIndex)
  ┌──────────────────────────────────────────┐
  │ 1. Load raw documents                    │
  │ 2. Clean + normalize (ภาษาไทย encoding)  │
  │ 3. Chunk (Semantic / Hierarchical)       │
  │ 4. Extract metadata อัตโนมัติ            │
  │ 5. Check content hash → skip if same    │
  │ 6. Embed (OpenAI / local model)          │
  │    └─ Embedding cache → Dragonfly       │
  │ 7. Store → ParadeDB                     │
  └──────────────────────────────────────────┘
         │
         ▼
[ParadeDB]
  documents table:
  - id, parent_doc_id
  - content (TEXT) → BM25 index
  - embedding (VECTOR 1536) → HNSW index
  - metadata (JSONB) → GIN index
  - content_hash, updated_at
         │
         ▼
[Query Pipeline]  (Python — LlamaIndex)
  User Question
    │
    ├─ 1. Semantic Cache Check (Dragonfly)
    │       similarity > 0.95 → return cached answer
    │
    ├─ 2. Embed question
    │       Embedding cache check (Dragonfly)
    │
    ├─ 3. Metadata filter (Permission check)
    │       access_level, department
    │
    ├─ 4. Hybrid Retrieve (ParadeDB)
    │       Vector Search (HNSW)
    │       BM25 Full-text Search
    │       RRF Fusion → top 10 candidates
    │
    ├─ 5. Rerank → top 3
    │       cross-encoder / Cohere Rerank
    │
    ├─ 6. Build prompt + context
    │
    ├─ 7. LLM (GPT-4o) → Answer
    │       Streaming support
    │
    └─ 8. Return { answer, sources, confidence }
              └─ Store to Semantic Cache

[API Layer]  (FastAPI)
  POST /query        → stream response
  POST /ingest       → trigger ingestion
  GET  /documents    → list docs
  DELETE /documents  → remove + re-index
  GET  /health
  WS   /stream

[Interfaces]
  Web Chat (Next.js/React)
  LINE Bot → Messaging API webhook → API Server
  Slack Bot → Slack Events API → API Server
  Cursor MCP → stdio/SSE → API Server
```

---

## 6. LlamaIndex Concepts ที่ต้องรู้

### 6.1 Core Objects

```python
# Document — หน่วยข้อมูลหลัก
from llama_index.core import Document

doc = Document(
    text="เนื้อหา...",
    metadata={"source": "hr.md", "dept": "hr"},
    id_="unique-doc-id",        # สำคัญ! ใช้ update/delete
    excluded_llm_metadata_keys=["internal_id"],  # metadata ไม่ส่งให้ LLM
)

# Node — chunk ที่แตกจาก Document
# Node มี relationships: PREVIOUS, NEXT, PARENT, CHILD, SOURCE
from llama_index.core.schema import TextNode, NodeRelationship, RelatedNodeInfo

node = TextNode(
    text="chunk content...",
    metadata={"source": "hr.md"},
    relationships={
        NodeRelationship.SOURCE: RelatedNodeInfo(node_id="parent-doc-id"),
    }
)
```

### 6.2 NodeParsers (Chunking)

```python
from llama_index.core.node_parser import (
    SentenceSplitter,           # แบ่งตาม sentence boundary
    SemanticSplitterNodeParser, # แบ่งตาม semantic similarity
    HierarchicalNodeParser,     # สร้าง parent-child hierarchy
    MarkdownNodeParser,         # เข้าใจ Markdown structure
    JSONNodeParser,             # สำหรับ JSON docs
)

# Hierarchical — แนะนำสำหรับ Sellsuki
from llama_index.core.node_parser import get_leaf_nodes, get_root_nodes

parser = HierarchicalNodeParser.from_defaults(
    chunk_sizes=[2048, 512, 128]
    # 2048 = parent (context ใหญ่ ส่งให้ LLM)
    # 512  = middle
    # 128  = leaf (search หา แล้วดึง parent ขึ้นมา)
)

nodes = parser.get_nodes_from_documents(docs)
leaf_nodes = get_leaf_nodes(nodes)  # ใช้ index เฉพาะ leaf
```

### 6.3 Query Modes

```python
# 1. Simple Vector Query
engine = index.as_query_engine(similarity_top_k=5)

# 2. With Filters
from llama_index.core.vector_stores import MetadataFilters, MetadataFilter, FilterOperator

engine = index.as_query_engine(
    filters=MetadataFilters(filters=[
        MetadataFilter(key="access_level", value="all"),
        MetadataFilter(key="department", value="hr", operator=FilterOperator.EQ),
    ])
)

# 3. HyDE (Hypothetical Document Embeddings)
# สร้าง hypothetical answer ก่อน แล้ว embed หา → recall สูงขึ้น
from llama_index.core.indices.query.query_transform.base import HyDEQueryTransform
from llama_index.core.query_engine import TransformQueryEngine

hyde = HyDEQueryTransform(include_original=True)
hyde_engine = TransformQueryEngine(base_engine, query_transform=hyde)

# 4. SubQuestion (ซับซ้อน — แบ่ง query ออกเป็น sub-queries)
from llama_index.core.query_engine import SubQuestionQueryEngine
from llama_index.core.tools import QueryEngineTool

tools = [
    QueryEngineTool.from_defaults(engine, name="hr_docs", description="HR policies"),
    QueryEngineTool.from_defaults(engine2, name="tech_docs", description="Technical docs"),
]
sub_engine = SubQuestionQueryEngine.from_defaults(query_engine_tools=tools)
# "ลาพักร้อนกี่วัน และ deploy ระบบยังไง?" → แยกถามทั้ง 2 ตัว

# 5. Router (เลือก engine ที่เหมาะสมอัตโนมัติ)
from llama_index.core.query_engine import RouterQueryEngine
from llama_index.core.selectors import LLMSingleSelector

router_engine = RouterQueryEngine(
    selector=LLMSingleSelector.from_defaults(),
    query_engine_tools=tools,
)
```

### 6.4 Response Modes

```python
# compact (default) — รวม context ให้กระทัดรัด
# tree_summarize — summarize หลายระดับ ดีสำหรับ doc ยาว
# refine — ตอบทีละ node แล้ว refine คำตอบ (ช้าแต่ละเอียด)
# simple_summarize — concatenate แล้ว summarize
# no_text — return nodes เฉย ๆ ไม่ให้ LLM ตอบ (debug)

engine = index.as_query_engine(response_mode="tree_summarize")
```

### 6.5 Streaming

```python
# Python streaming
engine = index.as_query_engine(streaming=True)
response = engine.query("ลาพักร้อนกี่วัน?")

for token in response.response_gen:
    print(token, end="", flush=True)

# FastAPI streaming endpoint
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

@app.post("/query/stream")
async def query_stream(question: str):
    async def generate():
        response = await engine.aquery(question)
        async for token in response.async_response_gen():
            yield f"data: {token}\n\n"
        yield "data: [DONE]\n\n"
    
    return StreamingResponse(generate(), media_type="text/event-stream")
```

---

## 7. Outline (Docs Sellsuki) Integration

```python
# Outline มี REST API — ต้องเขียน Custom Reader

import httpx
from llama_index.core.readers.base import BaseReader
from llama_index.core import Document

class OutlineReader(BaseReader):
    def __init__(self, base_url: str, api_token: str):
        self.base_url = base_url
        self.headers = {"Authorization": f"Bearer {api_token}"}
    
    async def load_all_documents(self) -> list[Document]:
        async with httpx.AsyncClient() as client:
            # ดึง collections ทั้งหมด
            cols = await client.post(
                f"{self.base_url}/api/collections.list",
                headers=self.headers,
                json={"limit": 100}
            )
            
            docs = []
            for col in cols.json()["data"]:
                # ดึง documents ใน collection
                doc_list = await client.post(
                    f"{self.base_url}/api/documents.list",
                    headers=self.headers,
                    json={"collectionId": col["id"], "limit": 100}
                )
                
                for doc_meta in doc_list.json()["data"]:
                    # ดึง content จริง
                    doc_detail = await client.post(
                        f"{self.base_url}/api/documents.info",
                        headers=self.headers,
                        json={"id": doc_meta["id"]}
                    )
                    d = doc_detail.json()["data"]
                    
                    docs.append(Document(
                        text=d["text"],  # Markdown content
                        metadata={
                            "source_id": d["id"],
                            "title": d["title"],
                            "collection": col["name"],
                            "url": d["url"],
                            "updated_at": d["updatedAt"],
                            "created_by": d["createdBy"]["name"],
                            # map collection → department/access_level
                            "department": self._map_collection(col["name"]),
                            "access_level": self._get_access(col["name"]),
                        },
                        id_=d["id"],
                    ))
            
            return docs
    
    def _map_collection(self, name: str) -> str:
        mapping = {
            "HR Policy": "hr",
            "Engineering": "engineering",
            "Product": "product",
            "General": "all",
        }
        return mapping.get(name, "general")
    
    def _get_access(self, collection_name: str) -> str:
        restricted = ["Finance", "Management", "Confidential"]
        return "restricted" if collection_name in restricted else "all"

# Webhook — เมื่อ Outline อัพเดต doc → trigger re-ingest
@app.post("/webhooks/outline")
async def outline_webhook(payload: dict):
    if payload["event"] in ["documents.update", "documents.create"]:
        doc_id = payload["payload"]["id"]
        # background task: re-ingest doc นี้
        background_tasks.add_task(reingest_document, doc_id)
    elif payload["event"] == "documents.delete":
        doc_id = payload["payload"]["id"]
        background_tasks.add_task(delete_document, doc_id)
```

---

## 8. Permission & Access Control

```python
# แต่ละ user มี role/department → filter ก่อน query

ROLE_ACCESS = {
    "employee": ["all"],
    "hr_staff": ["all", "hr"],
    "engineer": ["all", "engineering"],
    "manager": ["all", "hr", "engineering", "management"],
    "admin": ["all", "hr", "engineering", "management", "finance"],
}

def get_query_engine_for_user(user_role: str) -> BaseQueryEngine:
    allowed = ROLE_ACCESS.get(user_role, ["all"])
    
    filters = MetadataFilters(
        filters=[
            MetadataFilter(
                key="access_level",
                value=allowed,
                operator=FilterOperator.IN  # access_level IN allowed_list
            )
        ]
    )
    
    return index.as_query_engine(
        similarity_top_k=5,
        filters=filters,
        node_postprocessors=[reranker],
    )

# ใน API endpoint
@app.post("/query")
async def query(request: QueryRequest, user=Depends(get_current_user)):
    engine = get_query_engine_for_user(user.role)
    response = await engine.aquery(request.question)
    return {
        "answer": str(response),
        "sources": [
            {"title": n.metadata.get("title"), "url": n.metadata.get("url")}
            for n in response.source_nodes
        ]
    }
```

---

## 9. Evaluation — วัด Quality ยังไง

```python
# วิธีวัดว่า RAG ดีแค่ไหน

from llama_index.core.evaluation import (
    FaithfulnessEvaluator,   # คำตอบมาจาก context จริงไหม (ไม่ hallucinate)
    RelevancyEvaluator,      # context ที่ดึงมา relevant กับคำถามไหม
    CorrectnessEvaluator,    # คำตอบถูกต้องไหม (ต้องมี ground truth)
    BatchEvalRunner,
)

faithfulness = FaithfulnessEvaluator()
relevancy = RelevancyEvaluator()

# สร้าง test set
test_questions = [
    {"q": "ลาพักร้อนกี่วัน?", "expected": "10 วัน"},
    {"q": "WiFi password คืออะไร?", "expected": "..."},
]

# run evaluation
runner = BatchEvalRunner(
    {"faithfulness": faithfulness, "relevancy": relevancy},
    workers=4,
)

results = await runner.aevaluate_queries(
    query_engine, queries=[q["q"] for q in test_questions]
)

# metrics ที่ควร track
# Faithfulness Score  > 0.9  (คำตอบมาจาก doc จริง)
# Relevancy Score     > 0.8  (ดึง context ถูก)
# Answer Correctness  > 0.85 (คำตอบตรง ground truth)
# Latency P50         < 2s
# Latency P99         < 5s
# Cache Hit Rate      > 40%
```

---

## 10. LlamaIndex Workflows (Modern API — แทน Pipeline เก่า)

```python
# LlamaIndex >= 0.10 แนะนำใช้ Workflows แทน Chain
# เป็น event-driven, async-first, ง่าย debug

from llama_index.core.workflow import (
    Workflow, StartEvent, StopEvent,
    step, Event
)

class QueryEvent(Event):
    query: str
    user_role: str

class RetrievedEvent(Event):
    nodes: list
    query: str

class RAGWorkflow(Workflow):
    
    @step
    async def check_cache(self, ev: StartEvent) -> QueryEvent | StopEvent:
        cached = await semantic_cache.get(ev.query)
        if cached:
            return StopEvent(result=cached)
        return QueryEvent(query=ev.query, user_role=ev.user_role)
    
    @step
    async def retrieve(self, ev: QueryEvent) -> RetrievedEvent:
        engine = get_query_engine_for_user(ev.user_role)
        nodes = await retriever.aretrieve(ev.query)
        return RetrievedEvent(nodes=nodes, query=ev.query)
    
    @step
    async def rerank_and_generate(self, ev: RetrievedEvent) -> StopEvent:
        reranked = reranker.postprocess_nodes(ev.nodes, query_str=ev.query)
        response = await llm.apredict(
            prompt_template, 
            context="\n".join([n.text for n in reranked]),
            query=ev.query
        )
        await semantic_cache.set(ev.query, response)
        return StopEvent(result=response)

# ใช้งาน
workflow = RAGWorkflow(timeout=30)
result = await workflow.run(query="ลาพักร้อนกี่วัน?", user_role="employee")
```

---

## 11. Cost Estimation

### Embedding Cost (OpenAI text-embedding-3-small)

```
สมมติ Sellsuki มี docs ~500 หน้า
  500 หน้า × 500 tokens/หน้า = 250,000 tokens
  ราคา: $0.02 / 1M tokens
  ค่า embed ครั้งแรก = $0.005 (แทบฟรี)
  
  Incremental update (เดือนละ 50 หน้า) ≈ $0.0005/เดือน
```

### LLM Cost (GPT-4o)

```
สมมติพนักงาน 50 คน ถามวันละ 3 คำถาม = 150 queries/วัน
  Input: 150 × 2000 tokens (context + question) = 300,000 tokens
  Output: 150 × 300 tokens = 45,000 tokens
  
  GPT-4o: $2.50/1M input, $10/1M output
  ต่อวัน: (0.3 × $2.50) + (0.045 × $10) = $0.75 + $0.45 = $1.20/วัน
  ต่อเดือน: ~$36
  
  ถ้า cache hit 40%: ลดเหลือ ~$22/เดือน
```

### รวม Infrastructure

```
ParadeDB (EC2 t3.medium):  ~$30/เดือน
Dragonfly (cache):          ~$10/เดือน
API Server:                 ~$20/เดือน
OpenAI (LLM + Embed):       ~$25/เดือน
                           ─────────────
รวม:                        ~$85/เดือน
```

---

## 12. ข้อควรระวัง (Gotchas)

### ภาษาไทย
```python
# ❌ Default tokenizer ไม่เข้าใจภาษาไทย (ไม่มี space)
splitter = SentenceSplitter(chunk_size=512)  # อาจตัดกลางคำ

# ✅ ใช้ character-based + ปรับ separators
splitter = SentenceSplitter(
    chunk_size=512,
    chunk_overlap=64,
    paragraph_separator="\n\n",
    secondary_chunking_regex="[^ๆ๏\\u0E00-\\u0E7F]",  # Thai Unicode range
)

# ✅ หรือใช้ Thai tokenizer เช่น pythainlp
from pythainlp.tokenize import word_tokenize
```

### Embedding Model ภาษาไทย
```
OpenAI text-embedding-3-small  → รองรับไทยดี ✅
OpenAI text-embedding-3-large  → ดีกว่า แต่แพงกว่า 5x
BGE-M3 (local)                 → multilingual ดีมาก ฟรี แต่ต้องรัน server
```

### ParadeDB Notes
```sql
-- ParadeDB ใช้ pgvector extension
-- ต้อง enable ก่อน
CREATE EXTENSION IF NOT EXISTS vector;

-- BM25 ใช้ pg_search (ParadeDB's extension)
-- ต้อง enable
CREATE EXTENSION IF NOT EXISTS pg_search;

-- ภาษาไทย BM25 ต้องใช้ ICU tokenizer
-- default tokenizer ไม่รองรับไทย
```

### LlamaIndex Version
```
ใช้ >= 0.10.x (llamaindex-core)
ไม่ใช้ < 0.9.x (legacy, API เปลี่ยนไปเยอะมาก)

pip install llama-index-core
pip install llama-index-vector-stores-postgres
pip install llama-index-embeddings-openai
pip install llama-index-llms-openai
pip install llama-index-postprocessor-cohere-rerank
```

---

## 13. Recommended Final Stack

```
Language:     Python 3.11+  (RAG core)
              TypeScript     (Frontend / LINE / Slack bot)

RAG:          LlamaIndex Core >= 0.10

LLM:          GPT-4o (production)
              GPT-4o-mini (dev/cache miss fallback)

Embedding:    text-embedding-3-small (cost-effective)
              BGE-M3 (ถ้าต้องการ local + Thai ดีกว่า)

Vector DB:    ParadeDB (pgvector + BM25 hybrid)
Cache:        Dragonfly

API Server:   FastAPI + Uvicorn
Background:   ARQ (async job queue บน Redis/Dragonfly)

Doc Source:   Outline (Docs Sellsuki) via REST API

Observability: LlamaTrace / OpenTelemetry → Grafana
Testing:      pytest + LlamaIndex Evaluators

Deployment:   Docker Compose (dev)
              Railway / AWS ECS (production)
```

---

## 14. Quick Start — ทดลองใน 30 นาที

```python
# 1. Install
# pip install llama-index-core llama-index-embeddings-openai 
#             llama-index-llms-openai llama-index-vector-stores-postgres

# 2. ทดสอบกับ in-memory ก่อน (ไม่ต้องมี ParadeDB)
import os
os.environ["OPENAI_API_KEY"] = "sk-..."

from llama_index.core import VectorStoreIndex, Document, Settings
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.llms.openai import OpenAI

Settings.llm = OpenAI(model="gpt-4o-mini")
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")

# ใส่ข้อมูลทดสอบ
docs = [
    Document(text="พนักงาน Sellsuki มีสิทธิ์ลาพักร้อน 10 วันต่อปี หลังผ่านทดลองงาน 3 เดือน"),
    Document(text="WiFi สำนักงาน: SSID: Sellsuki-Corp, Password: sell@2025"),
    Document(text="การสร้าง Order ใน Sellsuki: ไปที่ Orders > Create New > กรอก customer info > เพิ่ม products > Submit"),
]

index = VectorStoreIndex.from_documents(docs)
engine = index.as_query_engine()

# ทดสอบ
questions = [
    "ลาพักร้อนได้กี่วัน?",
    "WiFi password คืออะไร?",
    "สร้าง order ยังไง?",
]

for q in questions:
    r = engine.query(q)
    print(f"Q: {q}")
    print(f"A: {r}\n")

# 3. เมื่อทดสอบแล้ว → ย้าย vector store ไป ParadeDB
# 4. เพิ่ม metadata, filter, reranker ทีละขั้น
```

---

*จัดทำสำหรับ Sellsuki RAG System | March 2026*
