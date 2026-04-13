# 🧠 LangChain, RAG & Sellsuki Knowledge Base — Deep Dive

---

## 📌 สารบัญ

1. [LangChain คืออะไร?](#1-langchain-คืออะไร)
2. [LangChain Ecosystem — ตระกูล Lang](#2-langchain-ecosystem--ตระกูล-lang)
3. [Core Concepts ที่ต้องรู้](#3-core-concepts-ที่ต้องรู้)
4. [RAG คืออะไร และทำงานยังไง?](#4-rag-คืออะไร-และทำงานยังไง)
5. [Chunking Strategies — ลึกมาก](#5-chunking-strategies--ลึกมาก)
6. [Metadata & Filtering](#6-metadata--filtering)
7. [Embedding & Vector DB (ParadeDB)](#7-embedding--vector-db-paradedb)
8. [Semantic Cache with Dragonfly](#8-semantic-cache-with-dragonfly)
9. [Smart Update — ไม่ Embed ซ้ำ](#9-smart-update--ไม่-embed-ซ้ำ)
10. [Database & Index Design for RAG](#10-database--index-design-for-rag)
11. [Full Architecture: Sellsuki RAG System](#11-full-architecture-sellsuki-rag-system)
12. [Code Examples แบบละเอียด](#12-code-examples-แบบละเอียด)
13. [Output ที่ได้จาก RAG](#13-output-ที่ได้จาก-rag)
14. [Deployment: LINE / Slack / Web / Cursor](#14-deployment-line--slack--web--cursor)

---

## 1. LangChain คืออะไร?

**LangChain** คือ Framework สำหรับสร้างแอปพลิเคชันที่ขับเคลื่อนด้วย LLM (Large Language Model)

### ปัญหาที่ LangChain แก้

```
ถ้าไม่มี LangChain:
- เรียก OpenAI API ตรงๆ → ได้แค่ Q&A ธรรมดา
- ต้องเขียน logic เชื่อม Tools เอง
- ต้องจัดการ Memory, Context Window เอง
- ต้องเขียน Retrieval Pipeline เองทุกอย่าง

มี LangChain:
- มี building blocks พร้อมใช้
- Chain หลาย steps เข้าด้วยกันได้
- รองรับ LLM หลาย provider
- มี ecosystem ครบ (Vector DB, Memory, Tools, Agents)
```

### LangChain ทำอะไรได้บ้าง?

| Feature | คำอธิบาย | ตัวอย่าง |
|---|---|---|
| **Chains** | เชื่อม steps หลายอันเข้าด้วยกัน | Load doc → Chunk → Embed → Query → Answer |
| **Agents** | ให้ AI ตัดสินใจเองว่าจะใช้ Tool ไหน | AI ค้น Google แล้วสรุป |
| **Memory** | จำบทสนทนา | Chat history ข้ามรอบ |
| **Retrievers** | ดึงข้อมูลจาก Vector DB | RAG Pipeline |
| **Tools** | ให้ AI ใช้ external service | Calculator, Search, API call |
| **Callbacks** | Hook เข้า lifecycle | Logging, Streaming, Monitoring |

---

## 2. LangChain Ecosystem — ตระกูล Lang

### 🔗 LangChain (Core)
- **คืออะไร:** Framework หลัก, มี Chains, Agents, Memory, Retrievers
- **ใช้เมื่อ:** สร้าง RAG, Chatbot, Workflow อัตโนมัติ
- **ภาษา:** Python, JavaScript/TypeScript
- **Output:** Text, JSON, Structured objects

```python
from langchain.chains import RetrievalQA
from langchain_openai import ChatOpenAI

chain = RetrievalQA.from_chain_type(
    llm=ChatOpenAI(model="gpt-4"),
    retriever=vectorstore.as_retriever()
)
result = chain.invoke("ลาพักร้อนกี่วัน?")
# Output: {"result": "พนักงานมีสิทธิ์ลาพักร้อน 10 วันต่อปี..."}
```

---

### 🔁 LangGraph
- **คืออะไร:** สร้าง Stateful, Multi-step Agents แบบ Graph/Flow
- **ใช้เมื่อ:** ต้องการ loop, conditional branching, Human-in-the-loop
- **ต่างจาก LangChain:** LangChain = linear chain | LangGraph = DAG (Directed Acyclic Graph)
- **Output:** State object ที่ไหลผ่าน nodes

```python
from langgraph.graph import StateGraph, END

# สร้าง workflow แบบ graph
workflow = StateGraph(AgentState)
workflow.add_node("retrieve", retrieve_documents)
workflow.add_node("grade_docs", grade_relevance)
workflow.add_node("generate", generate_answer)
workflow.add_node("web_search", fallback_search)

# Conditional edge: ถ้าเอกสารไม่เกี่ยวข้อง → ไป web_search
workflow.add_conditional_edges(
    "grade_docs",
    decide_next_step,
    {"relevant": "generate", "not_relevant": "web_search"}
)
```

**Use case ที่เหมาะ:** RAG ที่มี fallback logic เช่น ถ้าหาใน vector DB ไม่เจอ → ไป search web → ถ้ายังไม่มี → บอก HR

---

### 🔍 LangSmith
- **คืออะไร:** Platform สำหรับ Debug, Trace, Evaluate LLM apps
- **ใช้เมื่อ:** ต้องการดูว่า LLM คิดอะไร, ทำไมตอบผิด, ใช้เวลาเท่าไหร่
- **Output:** Dashboard แสดง traces, latency, token usage, evals

```python
# Enable LangSmith tracing
import os
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "ls_..."

# ทุก chain call จะถูก log อัตโนมัติ
# เห็น: Input → Retrieval → Prompt → LLM → Output
# เห็น: latency แต่ละ step, tokens used, cost
```

---

### 🌐 LangServe
- **คืออะไร:** Deploy LangChain chains เป็น REST API อัตโนมัติ
- **ใช้เมื่อ:** ต้องการ expose chain เป็น endpoint ให้ LINE/Slack/Web เรียก
- **Output:** FastAPI server พร้อม `/invoke`, `/stream`, `/batch` endpoints

```python
from langserve import add_routes
from fastapi import FastAPI

app = FastAPI()
add_routes(app, sellsuki_rag_chain, path="/sellsuki-rag")

# ได้ endpoint:
# POST /sellsuki-rag/invoke   → ตอบทันที
# POST /sellsuki-rag/stream   → stream คำตอบ
# GET  /sellsuki-rag/playground → UI ทดสอบ
```

---

### 🧩 LangChain Hub
- **คืออะไร:** Repository ของ Prompts สำเร็จรูป
- **ใช้เมื่อ:** ต้องการ prompt ที่ผ่านการ test มาแล้ว

```python
from langchain import hub
# ดึง RAG prompt สำเร็จรูป
prompt = hub.pull("rlm/rag-prompt")
```

---

### เปรียบเทียบ Lang ต่างๆ

| Framework | หน้าที่ | Output |
|---|---|---|
| **LangChain** | Core building blocks | Chains, Agents, Retrievers |
| **LangGraph** | Complex workflows | Stateful graph execution |
| **LangSmith** | Observability | Traces, Metrics, Evals |
| **LangServe** | Deployment | REST API |
| **LangChain Hub** | Prompt sharing | Reusable prompts |

---

## 3. Core Concepts ที่ต้องรู้

### 📄 Document
```python
from langchain.schema import Document

doc = Document(
    page_content="พนักงานมีสิทธิ์ลาพักร้อน 10 วันต่อปี",
    metadata={
        "source": "hr_policy.pdf",
        "page": 3,
        "department": "HR",
        "last_updated": "2024-01-15",
        "access_level": "all_staff"
    }
)
```

### 🔗 Chain (LCEL — LangChain Expression Language)
```python
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

# LCEL syntax: component | component | component
chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

# เรียกใช้
answer = chain.invoke("ลาพักร้อนกี่วัน?")
```

### 🤖 Agent vs Chain

```
Chain:  คำถาม → Step1 → Step2 → Step3 → คำตอบ  (กำหนดไว้ล่วงหน้า)
Agent:  คำถาม → AI คิด → เลือก Tool → ดูผล → คิดอีก → คำตอบ  (dynamic)
```

---

## 4. RAG คืออะไร และทำงานยังไง?

**RAG = Retrieval-Augmented Generation**

แทนที่จะให้ LLM ตอบจากความรู้ที่ฝึกมา (อาจเก่า/ผิด/เดา) → ให้ดึงเอกสารจริงมาก่อน แล้วค่อยตอบ

### RAG Pipeline แบบละเอียด

```
Phase 1: INDEXING (ทำครั้งเดียว หรือเมื่อมี doc ใหม่)
┌─────────────────────────────────────────────────────┐
│  Documents                                          │
│  (PDF, DOCX, Notion, Confluence, Web)               │
│       ↓                                             │
│  Document Loader                                    │
│       ↓                                             │
│  Text Splitter (Chunking)                           │
│  + Metadata tagging                                 │
│       ↓                                             │
│  Embedding Model                                    │
│  (text → vector [1536 dimensions])                  │
│       ↓                                             │
│  Vector Store (ParadeDB)                            │
│  + Hash store (Dragonfly) สำหรับ dedup             │
└─────────────────────────────────────────────────────┘

Phase 2: QUERYING (ทุกครั้งที่ user ถาม)
┌─────────────────────────────────────────────────────┐
│  User Question: "ลาพักร้อนกี่วัน?"                  │
│       ↓                                             │
│  Check Semantic Cache (Dragonfly)                   │
│  คำถามคล้ายกันเคยถามแล้วไหม?                        │
│       ↓ (cache miss)                                │
│  Embed Question → Query Vector                      │
│       ↓                                             │
│  Vector Search in ParadeDB                          │
│  + Metadata Filter (เช่น department=HR)            │
│       ↓                                             │
│  Top-K Relevant Chunks (k=5)                        │
│       ↓                                             │
│  Reranking (เรียงใหม่ให้เกี่ยวข้องที่สุด)           │
│       ↓                                             │
│  Prompt Construction:                               │
│  "Context: [chunks] \n Question: [question]"        │
│       ↓                                             │
│  LLM Generate Answer                               │
│       ↓                                             │
│  Save to Semantic Cache                             │
│       ↓                                             │
│  Return Answer + Sources                            │
└─────────────────────────────────────────────────────┘
```

---

## 5. Chunking Strategies — ลึกมาก

Chunking คือการตัดเอกสารใหญ่เป็นชิ้นเล็กๆ เพื่อให้ Embed และ Retrieve ได้แม่นยำ

### Strategy 1: Fixed-Size Chunking (ง่ายสุด)

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,      # ขนาด chunk (characters)
    chunk_overlap=50,    # overlap เพื่อไม่ให้ context ขาด
    separators=["\n\n", "\n", ".", " "]
)

chunks = splitter.split_documents(docs)
```

**เหมาะกับ:** เอกสารทั่วไป, เริ่มต้น  
**ข้อเสีย:** ตัดกลางประโยคได้, ไม่เข้าใจ structure

---

### Strategy 2: Semantic Chunking (ฉลาดขึ้น)

แทนที่จะตัดตามขนาด → ตัดเมื่อความหมายเปลี่ยน

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

splitter = SemanticChunker(
    OpenAIEmbeddings(),
    breakpoint_threshold_type="percentile",  # ตัดเมื่อ similarity ต่ำกว่า percentile
    breakpoint_threshold_amount=95
)

chunks = splitter.split_documents(docs)
# ผลลัพธ์: แต่ละ chunk จะพูดถึงเรื่องเดียวกันทั้งหมด
```

**เหมาะกับ:** HR Policy, Technical Docs  
**ตัวอย่าง Output:**
```
Chunk 1: [ทุกอย่างเกี่ยวกับวันลา]
Chunk 2: [ทุกอย่างเกี่ยวกับสวัสดิการ]
Chunk 3: [ทุกอย่างเกี่ยวกับการประเมินงาน]
```

---

### Strategy 3: Parent-Child Chunking (ดีที่สุดสำหรับ RAG)

**ปัญหา:** chunk เล็ก = retrieve แม่น, แต่ context น้อย | chunk ใหญ่ = context ดี แต่ retrieve ไม่แม่น  
**แก้:** Embed chunk เล็ก (เพื่อ retrieve) แต่ return parent chunk ใหญ่ (เพื่อ context)

```python
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore
from langchain.text_splitter import RecursiveCharacterTextSplitter

# Parent: chunks ใหญ่ (2000 chars) — ให้ LLM อ่าน
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=2000)

# Child: chunks เล็ก (200 chars) — ใช้ embed และ search
child_splitter = RecursiveCharacterTextSplitter(chunk_size=200)

store = InMemoryStore()  # เก็บ parent chunks
retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,
    docstore=store,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)

retriever.add_documents(docs)

# เวลา query:
# 1. embed คำถาม → search ใน child chunks (แม่น)
# 2. return parent chunk ของ child ที่เจอ (context ดี)
```

---

### Strategy 4: Proposition Chunking (ล้ำที่สุด)

แทนที่จะ chunk ตามขนาด → ให้ LLM แปลงแต่ละ chunk เป็น "ข้อความที่ตอบได้ด้วยตัวเอง"

```python
proposition_prompt = """
แปลงข้อความต่อไปนี้เป็นประโยคสั้นๆ ที่สมบูรณ์ในตัวเอง
แต่ละประโยคต้องตอบคำถามได้โดยไม่ต้องอ่าน context อื่น

ข้อความ: {text}

ตัวอย่าง:
Input: "พนักงานที่ทำงานครบ 1 ปี มีสิทธิ์ลาพักร้อน 10 วัน และสามารถสะสมได้ไม่เกิน 20 วัน"
Output:
- พนักงานที่ทำงานครบ 1 ปีมีสิทธิ์ลาพักร้อน 10 วันต่อปี
- วันลาพักร้อนสะสมได้สูงสุด 20 วัน
- สิทธิ์ลาพักร้อนเริ่มหลังทำงานครบ 1 ปี
"""
```

**เหมาะกับ:** FAQ systems, HR chatbot  
**Output:** แต่ละ chunk ตอบคำถามได้ชัดเจน ไม่ต้องอ่าน context เพิ่ม

---

### Strategy 5: Document-Specific Chunking

```python
# สำหรับ Markdown (เช่น Confluence/Notion docs)
from langchain.text_splitter import MarkdownHeaderTextSplitter

headers = [
    ("#", "chapter"),
    ("##", "section"),
    ("###", "subsection"),
]
splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers)
chunks = splitter.split_text(markdown_text)
# metadata อัตโนมัติ: {"chapter": "HR Policy", "section": "Leave Policy"}

# สำหรับ Code
from langchain.text_splitter import Language, RecursiveCharacterTextSplitter
code_splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.PYTHON,
    chunk_size=1000
)
# ตัดตาม function/class ไม่ตัดกลาง code block
```

---

### สรุปการเลือก Chunking Strategy สำหรับ Sellsuki

| เอกสาร | Strategy | เหตุผล |
|---|---|---|
| HR Policy PDF | Parent-Child + Semantic | แม่นยำ + context ครบ |
| Technical Docs / API Docs | Markdown Header | รักษา structure |
| Source Code | Language-aware | ไม่ตัดกลาง function |
| FAQ | Proposition | แต่ละ chunk ตอบได้เลย |
| Onboarding guides | Parent-Child | context ยาว แต่ search แม่น |

---

## 6. Metadata & Filtering

Metadata คือข้อมูลเสริมที่แนบมากับ chunk ใช้สำหรับ filter, search, และจำกัดสิทธิ์

### Metadata Schema สำหรับ Sellsuki

```python
metadata = {
    # 📂 Source info
    "source": "hr_policy_2024.pdf",
    "source_type": "pdf",           # pdf, notion, confluence, gdrive
    "page": 5,
    "url": "https://docs.sellsuki.com/hr/leave-policy",

    # 🏷️ Content classification
    "department": "HR",             # HR, Engineering, Finance, All
    "category": "leave_policy",     # leave, benefit, wifi, process, code
    "doc_type": "policy",           # policy, guide, faq, api_doc, runbook

    # 🔒 Access control
    "access_level": "all_staff",    # all_staff, manager, exec, engineering
    "confidential": False,

    # 🕐 Freshness
    "created_at": "2024-01-01",
    "last_updated": "2024-11-15",
    "version": "2.1",
    "is_outdated": False,

    # 🔑 Deduplication
    "chunk_hash": "sha256:abc123...",  # สำหรับ smart update
    "doc_id": "hr_policy_v2.1",
    "chunk_index": 3
}
```

### การใช้ Metadata Filter ตอน Query

```python
# กรณี 1: พนักงานทั่วไปถาม
retriever = vectorstore.as_retriever(
    search_kwargs={
        "k": 5,
        "filter": {
            "access_level": {"$in": ["all_staff"]},
            "is_outdated": False
        }
    }
)

# กรณี 2: Manager ถาม (เห็นได้มากกว่า)
retriever = vectorstore.as_retriever(
    search_kwargs={
        "filter": {
            "access_level": {"$in": ["all_staff", "manager"]},
            "is_outdated": False
        }
    }
)

# กรณี 3: Dev ถามเรื่อง code
retriever = vectorstore.as_retriever(
    search_kwargs={
        "filter": {
            "department": {"$in": ["Engineering", "All"]},
            "doc_type": {"$in": ["api_doc", "runbook", "guide"]}
        }
    }
)

# กรณี 4: ถามแบบ hybrid (semantic + keyword)
results = vectorstore.similarity_search(
    query="สร้าง order ยังไง",
    k=5,
    filter={"category": "process"}
)
```

---

## 7. Embedding & Vector DB (ParadeDB)

### Embedding คืออะไร?

```
ข้อความ → Embedding Model → Vector (array ของตัวเลข)

"ลาพักร้อนกี่วัน?"  → [0.12, -0.34, 0.87, ..., 0.23]  (1536 numbers)
"annual leave days?" → [0.11, -0.36, 0.85, ..., 0.21]  (คล้ายกันมาก!)
"สวัสดีครับ"         → [-0.67, 0.12, -0.45, ..., 0.88]  (ต่างกันมาก)

ยิ่ง vector ใกล้กัน = ความหมายใกล้กัน
วัดด้วย Cosine Similarity
```

### ParadeDB

ParadeDB คือ PostgreSQL extension ที่รองรับ:
- **pgvector:** Vector similarity search
- **BM25 full-text search:** keyword search แบบ Elasticsearch
- **Hybrid search:** รวม vector + keyword ในคำสั่งเดียว

```sql
-- สร้าง table สำหรับ RAG
CREATE TABLE sellsuki_docs (
    id          SERIAL PRIMARY KEY,
    chunk_text  TEXT NOT NULL,
    embedding   vector(1536),        -- pgvector
    chunk_hash  VARCHAR(64) UNIQUE,  -- สำหรับ dedup
    
    -- Metadata columns (สำหรับ filter เร็ว)
    source          TEXT,
    department      TEXT,
    access_level    TEXT,
    doc_type        TEXT,
    category        TEXT,
    last_updated    DATE,
    is_outdated     BOOLEAN DEFAULT FALSE,
    doc_id          TEXT,
    chunk_index     INT,
    raw_metadata    JSONB             -- metadata ที่เหลือ
);

-- Index สำหรับ vector search
CREATE INDEX ON sellsuki_docs 
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- Index สำหรับ BM25 (full-text)
CREATE INDEX ON sellsuki_docs 
USING bm25 (chunk_text, department, category);

-- Index สำหรับ metadata filter
CREATE INDEX ON sellsuki_docs (department, access_level, is_outdated);
CREATE INDEX ON sellsuki_docs (chunk_hash);  -- สำหรับ dedup check
```

### Hybrid Search Query

```sql
-- Hybrid search: รวม vector similarity + BM25 keyword
SELECT 
    id,
    chunk_text,
    source,
    department,
    -- Vector score
    1 - (embedding <=> $1::vector) AS vector_score,
    -- BM25 score  
    paradedb.score(id) AS bm25_score,
    -- Combined score (RRF: Reciprocal Rank Fusion)
    (0.7 * (1 - (embedding <=> $1::vector))) + 
    (0.3 * paradedb.score(id)) AS combined_score
FROM sellsuki_docs
WHERE 
    access_level = ANY($2)      -- permission filter
    AND is_outdated = FALSE
    AND chunk_text @@@ $3       -- BM25 match
ORDER BY combined_score DESC
LIMIT 5;
```

### LangChain + ParadeDB

```python
from langchain_community.vectorstores import PGVector
from langchain_openai import OpenAIEmbeddings

embedding_model = OpenAIEmbeddings(model="text-embedding-3-small")

vectorstore = PGVector(
    connection_string="postgresql://user:pass@localhost/sellsuki_rag",
    embedding_function=embedding_model,
    collection_name="sellsuki_docs",
    use_jsonb=True
)

# Add documents
vectorstore.add_documents(chunks)

# Search with filter
results = vectorstore.similarity_search_with_score(
    query="ลาพักร้อนกี่วัน",
    k=5,
    filter={"access_level": "all_staff", "is_outdated": False}
)
```

---

## 8. Semantic Cache with Dragonfly

### ทำไมต้องมี Cache?

```
ไม่มี Cache:
พนักงาน 100 คน ถามว่า "ลาพักร้อนกี่วัน?" ในวันเดียวกัน
→ Embed 100 ครั้ง → Search Vector DB 100 ครั้ง → Call LLM 100 ครั้ง
→ ช้า, แพง, ไม่จำเป็น

มี Cache:
ครั้งแรก: process ปกติ → เก็บ cache
ครั้งที่ 2-100: ถามคล้ายกัน → return จาก cache ทันที (< 5ms)
```

### 3 ระดับของ Cache

```
Level 1: Exact Cache (Redis/Dragonfly)
"ลาพักร้อนกี่วัน" == "ลาพักร้อนกี่วัน" → hit ทันที

Level 2: Semantic Cache (Dragonfly + Embedding)
"ลาพักร้อนกี่วัน" ≈ "สิทธิ์ลาพักร้อนมีกี่วัน?" → hit ถ้า similarity > 0.95

Level 3: Embedding Cache (Dragonfly)
เก็บ vector ของ query ที่เคย embed ไว้แล้ว
ถ้า query เหมือนเดิม → ไม่ต้อง embed ซ้ำ
```

### Dragonfly Implementation

```python
import dragonfly
import hashlib
import json
import numpy as np
from langchain_openai import OpenAIEmbeddings

class SellsukiCacheSystem:
    def __init__(self):
        self.cache = dragonfly.Redis(host="localhost", port=6379)
        self.embedder = OpenAIEmbeddings(model="text-embedding-3-small")
        self.SEMANTIC_THRESHOLD = 0.95  # cosine similarity cutoff
        self.CACHE_TTL = 3600 * 24      # 24 hours
        self.EMBED_CACHE_TTL = 3600 * 24 * 7  # 7 days

    # ---- Embedding Cache ----
    def get_or_create_embedding(self, text: str) -> list[float]:
        """Cache embeddings เพื่อไม่ต้อง call API ซ้ำ"""
        text_hash = hashlib.sha256(text.encode()).hexdigest()
        cache_key = f"embed:{text_hash}"

        cached = self.cache.get(cache_key)
        if cached:
            return json.loads(cached)  # Cache hit

        embedding = self.embedder.embed_query(text)  # API call
        self.cache.setex(cache_key, self.EMBED_CACHE_TTL, json.dumps(embedding))
        return embedding

    # ---- Semantic Cache ----
    def cosine_similarity(self, v1: list, v2: list) -> float:
        a, b = np.array(v1), np.array(v2)
        return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

    def get_semantic_cache(self, query: str, user_role: str) -> dict | None:
        """ค้นหา cache ที่มี query คล้ายกัน"""
        query_vec = self.get_or_create_embedding(query)
        
        # ดึง cached queries ทั้งหมด (ในระบบจริงใช้ vector search)
        cache_index_key = f"semantic_index:{user_role}"
        cached_items = self.cache.lrange(cache_index_key, 0, -1)
        
        best_match = None
        best_score = 0
        
        for item_key in cached_items:
            item = json.loads(self.cache.get(item_key) or "{}")
            if not item:
                continue
            sim = self.cosine_similarity(query_vec, item["query_embedding"])
            if sim > self.SEMANTIC_THRESHOLD and sim > best_score:
                best_score = sim
                best_match = item

        return best_match  # None = cache miss

    def set_semantic_cache(self, query: str, answer: str, sources: list, user_role: str):
        """เก็บ query+answer ลง semantic cache"""
        query_vec = self.get_or_create_embedding(query)
        query_hash = hashlib.sha256(query.encode()).hexdigest()[:16]
        
        cache_key = f"sem:{user_role}:{query_hash}"
        cache_data = {
            "query": query,
            "query_embedding": query_vec,
            "answer": answer,
            "sources": sources,
            "cached_at": "2024-11-15T10:00:00"
        }
        
        self.cache.setex(cache_key, self.CACHE_TTL, json.dumps(cache_data))
        self.cache.lpush(f"semantic_index:{user_role}", cache_key)
        self.cache.ltrim(f"semantic_index:{user_role}", 0, 999)  # เก็บ 1000 entries
```

---

## 9. Smart Update — ไม่ Embed ซ้ำ

### ปัญหา
```
Document เดิม: hr_policy.pdf (100 pages, 500 chunks)
มีการ update หน้า 5 เพียงหน้าเดียว

Naive approach: ลบทั้งหมด → embed ใหม่ทั้งหมด 500 chunks
→ เสียเงิน, เสียเวลา, ไม่จำเป็น

Smart approach: hash แต่ละ chunk → update เฉพาะ chunk ที่เปลี่ยน
```

### Content-Addressed Chunking

```python
import hashlib
from dataclasses import dataclass

@dataclass
class SmartChunk:
    text: str
    metadata: dict
    hash: str  # SHA-256 ของ content

def compute_chunk_hash(text: str, metadata: dict) -> str:
    """Hash ของ content + metadata สำคัญ"""
    content = text + metadata.get("source", "") + metadata.get("version", "")
    return hashlib.sha256(content.encode()).hexdigest()

class SmartDocumentIndexer:
    def __init__(self, vectorstore, cache: SellsukiCacheSystem, db_conn):
        self.vectorstore = vectorstore
        self.cache = cache
        self.db = db_conn

    def get_existing_hashes(self, doc_id: str) -> set[str]:
        """ดึง hash ของ chunks ที่มีอยู่แล้วใน DB"""
        result = self.db.execute(
            "SELECT chunk_hash FROM sellsuki_docs WHERE doc_id = %s",
            (doc_id,)
        ).fetchall()
        return {row[0] for row in result}

    def index_document(self, doc_path: str, metadata: dict):
        """Index document แบบ smart — ไม่ embed ซ้ำ"""
        
        # 1. Load & chunk document
        raw_chunks = self.load_and_chunk(doc_path)
        doc_id = metadata["doc_id"]
        
        # 2. ดึง hash เดิมที่มีใน DB
        existing_hashes = self.get_existing_hashes(doc_id)
        
        # 3. แยก chunks ใหม่ vs เดิม
        new_chunks = []
        current_hashes = set()
        
        for chunk in raw_chunks:
            chunk_hash = compute_chunk_hash(chunk.page_content, metadata)
            current_hashes.add(chunk_hash)
            chunk.metadata["chunk_hash"] = chunk_hash
            chunk.metadata["doc_id"] = doc_id
            
            if chunk_hash not in existing_hashes:
                new_chunks.append(chunk)  # ต้อง embed ใหม่
        
        # 4. หา chunks ที่หายไป (ต้อง delete)
        deleted_hashes = existing_hashes - current_hashes
        
        # 5. Delete chunks เก่าที่ไม่มีแล้ว
        if deleted_hashes:
            placeholders = ",".join(["%s"] * len(deleted_hashes))
            self.db.execute(
                f"DELETE FROM sellsuki_docs WHERE chunk_hash IN ({placeholders})",
                tuple(deleted_hashes)
            )
            print(f"🗑️ Deleted {len(deleted_hashes)} outdated chunks")
        
        # 6. Embed & Insert เฉพาะ chunks ใหม่
        if new_chunks:
            self.vectorstore.add_documents(new_chunks)
            print(f"✅ Added {len(new_chunks)} new chunks")
        else:
            print("✨ No changes detected — skipping embedding")

        # 7. Invalidate semantic cache สำหรับ doc นี้
        if new_chunks or deleted_hashes:
            self.invalidate_cache_for_doc(doc_id)
        
        summary = {
            "total_chunks": len(raw_chunks),
            "new_chunks": len(new_chunks),
            "deleted_chunks": len(deleted_hashes),
            "unchanged_chunks": len(raw_chunks) - len(new_chunks),
            "embeddings_saved": len(raw_chunks) - len(new_chunks)
        }
        return summary
```

### ตัวอย่าง Output ของ Smart Update

```
📄 Processing: hr_policy_v2.1.pdf
──────────────────────────────────
✅ Added 3 new chunks (หน้า 5 ที่เปลี่ยน)
🗑️ Deleted 2 outdated chunks  
✨ 495 chunks unchanged — skipped embedding

💰 Cost saved: 495 × $0.0001 = $0.0495 (per update)
⚡ Time saved: ~45 seconds
```

---

## 10. Database & Index Design for RAG

### Schema ครบ

```sql
-- Main documents table
CREATE TABLE sellsuki_docs (
    id              BIGSERIAL PRIMARY KEY,
    chunk_text      TEXT NOT NULL,
    embedding       vector(1536),
    chunk_hash      VARCHAR(64) UNIQUE NOT NULL,
    
    -- Source
    source          TEXT,
    source_type     VARCHAR(50),   -- pdf, notion, confluence, gdrive
    page_number     INT,
    url             TEXT,
    
    -- Classification  
    department      VARCHAR(50),
    category        VARCHAR(100),
    doc_type        VARCHAR(50),
    
    -- Access
    access_level    VARCHAR(50) DEFAULT 'all_staff',
    
    -- Versioning
    doc_id          VARCHAR(200),
    chunk_index     INT,
    last_updated    TIMESTAMPTZ DEFAULT NOW(),
    is_outdated     BOOLEAN DEFAULT FALSE,
    
    -- Extra
    raw_metadata    JSONB,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Vector index (HNSW = faster search, IVFFlat = less memory)
CREATE INDEX idx_embedding_hnsw ON sellsuki_docs 
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Filter indexes
CREATE INDEX idx_dept_access ON sellsuki_docs (department, access_level);
CREATE INDEX idx_doc_id ON sellsuki_docs (doc_id);
CREATE INDEX idx_hash ON sellsuki_docs (chunk_hash);
CREATE INDEX idx_outdated ON sellsuki_docs (is_outdated) WHERE is_outdated = FALSE;

-- Full-text search (BM25 via ParadeDB)
CREATE INDEX idx_bm25 ON sellsuki_docs 
USING bm25 (id, chunk_text, department, category, doc_type)
WITH (key_field = 'id');
```

### Query Performance

```sql
-- Plan สำหรับ hybrid search พร้อม filter
EXPLAIN ANALYZE
SELECT id, chunk_text, source,
    1 - (embedding <=> '[...]'::vector) AS similarity
FROM sellsuki_docs
WHERE 
    access_level = ANY(ARRAY['all_staff', 'manager'])
    AND is_outdated = FALSE
    AND department IN ('HR', 'All')
ORDER BY embedding <=> '[...]'::vector
LIMIT 5;

-- Expected: Index Scan on idx_dept_access → Filter → HNSW scan
-- Target latency: < 50ms
```

---

## 11. Full Architecture: Sellsuki RAG System

```
┌─────────────────────────────────────────────────────────────────┐
│                    SELLSUKI KNOWLEDGE BASE                       │
│                                                                  │
│  Data Sources          Processing Pipeline      Storage          │
│  ┌──────────┐         ┌─────────────────┐      ┌─────────────┐ │
│  │ Notion   │────────▶│ Document Loader │      │  ParadeDB   │ │
│  │ Confluence│        │ (LangChain)     │      │  (pgvector  │ │
│  │ Google   │         │       ↓         │─────▶│  + BM25)    │ │
│  │ Drive    │         │ Smart Chunker   │      └─────────────┘ │
│  │ PDF/DOCX │         │       ↓         │                       │
│  └──────────┘         │ Hash Check      │      ┌─────────────┐ │
│                       │ (no dup embed)  │      │  Dragonfly  │ │
│                       │       ↓         │─────▶│  - Sem Cache│ │
│                       │ Embed + Metadata│      │  - Emb Cache│ │
│                       └─────────────────┘      └─────────────┘ │
│                                                                  │
│  Query Pipeline                                                  │
│  ┌────────┐   ┌──────────┐   ┌─────────┐   ┌──────────────┐   │
│  │ User   │──▶│ Auth &   │──▶│Semantic │──▶│ Vector Search│   │
│  │Question│   │ Role     │   │ Cache   │   │ + Metadata   │   │
│  └────────┘   │ Detection│   │ Check   │   │ Filter       │   │
│               └──────────┘   └─────────┘   └──────┬───────┘   │
│                                                    ↓            │
│               ┌──────────────────────────────────────────────┐ │
│               │ LLM (Claude/GPT-4) + Prompt + Context        │ │
│               └──────────────────────┬───────────────────────┘ │
│                                      ↓                          │
│  Interfaces    ┌────────────────────────────────────────────┐  │
│  LINE Bot ────▶│                                            │  │
│  Slack Bot ───▶│    Structured Answer + Sources + Metadata  │  │
│  Web Chat ────▶│                                            │  │
│  Cursor ──────▶└────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 12. Code Examples แบบละเอียด

### Full RAG Chain

```python
from langchain_core.runnables import RunnablePassthrough, RunnableLambda
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

# Prompt template
SELLSUKI_RAG_PROMPT = ChatPromptTemplate.from_template("""
คุณคือ AI Assistant ของบริษัท Sellsuki ที่ตอบคำถามจากเอกสารบริษัทเท่านั้น

กฎการตอบ:
1. ตอบจากข้อมูลใน Context เท่านั้น ห้ามเดาหรืออ้างอิงความรู้นอก context
2. ถ้าไม่มีข้อมูลใน context → บอกว่า "ไม่พบข้อมูลในเอกสาร กรุณาติดต่อ HR โดยตรง"
3. อ้างอิงแหล่งที่มาเสมอ
4. ตอบเป็นภาษาไทย ชัดเจน กระชับ

Context จากเอกสาร:
{context}

คำถาม: {question}

คำตอบ (พร้อมอ้างอิง):
""")

def format_docs_with_sources(docs) -> str:
    formatted = []
    for i, doc in enumerate(docs, 1):
        source = doc.metadata.get("source", "Unknown")
        page = doc.metadata.get("page", "")
        page_str = f" หน้า {page}" if page else ""
        formatted.append(f"[{i}] {doc.page_content}\n(แหล่งที่มา: {source}{page_str})")
    return "\n\n".join(formatted)

def build_rag_chain(user_role: str = "all_staff"):
    llm = ChatOpenAI(model="gpt-4o", temperature=0)
    
    retriever = vectorstore.as_retriever(
        search_type="similarity",
        search_kwargs={
            "k": 5,
            "filter": {
                "access_level": {"$in": [user_role, "all_staff"]},
                "is_outdated": False
            }
        }
    )
    
    chain = (
        {
            "context": retriever | RunnableLambda(format_docs_with_sources),
            "question": RunnablePassthrough()
        }
        | SELLSUKI_RAG_PROMPT
        | llm
        | StrOutputParser()
    )
    return chain

# ใช้งาน
chain = build_rag_chain(user_role="all_staff")
answer = chain.invoke("ลาพักร้อนกี่วัน?")
```

### Full Query Handler พร้อม Cache

```python
async def handle_query(question: str, user_id: str, user_role: str) -> dict:
    cache = SellsukiCacheSystem()
    
    # Step 1: Check semantic cache
    cached = cache.get_semantic_cache(question, user_role)
    if cached:
        return {
            "answer": cached["answer"],
            "sources": cached["sources"],
            "from_cache": True,
            "latency_ms": 3  # near-instant
        }
    
    # Step 2: Build chain with role-based retriever
    chain = build_rag_chain(user_role=user_role)
    
    # Step 3: Get answer
    import time
    start = time.time()
    answer = chain.invoke(question)
    latency = int((time.time() - start) * 1000)
    
    # Step 4: Get sources (for citation)
    retriever = vectorstore.as_retriever(
        search_kwargs={"k": 5, "filter": {"access_level": user_role}}
    )
    source_docs = retriever.get_relevant_documents(question)
    sources = [
        {"file": d.metadata.get("source"), "page": d.metadata.get("page")}
        for d in source_docs
    ]
    
    # Step 5: Save to cache
    cache.set_semantic_cache(question, answer, sources, user_role)
    
    return {
        "answer": answer,
        "sources": sources,
        "from_cache": False,
        "latency_ms": latency
    }
```

---

## 13. Output ที่ได้จาก RAG

### ตัวอย่าง Output จริงๆ

**Input:** "ลาพักร้อนกี่วัน?"

```json
{
  "answer": "พนักงาน Sellsuki มีสิทธิ์ลาพักร้อน 10 วันต่อปี โดยมีเงื่อนไขดังนี้:\n- เริ่มสะสมสิทธิ์หลังทำงานครบ 1 ปี\n- สะสมข้ามปีได้สูงสุด 20 วัน\n- ต้องแจ้งล่วงหน้าอย่างน้อย 3 วันทำการ\n- อนุมัติโดย Direct Manager",
  "sources": [
    {"file": "hr_policy_2024.pdf", "page": 12, "section": "Leave Policy"},
    {"file": "employee_handbook.pdf", "page": 5, "section": "Benefits"}
  ],
  "from_cache": false,
  "latency_ms": 1240,
  "confidence": "high",
  "metadata": {
    "chunks_retrieved": 3,
    "top_similarity_score": 0.94
  }
}
```

**Input:** "WiFi password คืออะไร?"

```json
{
  "answer": "WiFi สำนักงานกรุงเทพ:\n- ชื่อ Network: Sellsuki-Office\n- Password: Ask IT department\n\n(หมายเหตุ: password ถูก redact เพื่อความปลอดภัย กรุณาติดต่อ IT Support ที่ it@sellsuki.com)",
  "sources": [
    {"file": "office_guide.pdf", "page": 2, "section": "IT Setup"}
  ],
  "from_cache": true,
  "latency_ms": 4
}
```

**Input:** "สร้าง order ใน Sellsuki Platform ยังไง?"

```json
{
  "answer": "ขั้นตอนการสร้าง Order ใน Sellsuki Platform:\n\n1. เข้าสู่ระบบที่ app.sellsuki.com\n2. ไปที่เมนู Orders → New Order\n3. เลือก Channel (Shopee/Lazada/LINE)\n4. กรอกข้อมูลลูกค้า\n5. เพิ่มสินค้าจาก Catalog\n6. ตรวจสอบ Stock availability\n7. กด Confirm Order\n\nหมายเหตุ: Order จะถูก sync ไปยัง ERP ภายใน 5 นาที",
  "sources": [
    {"file": "platform_user_guide_v3.pdf", "page": 45},
    {"file": "order_management_sop.pdf", "page": 8}
  ],
  "from_cache": false,
  "latency_ms": 1580
}
```

---

## 14. Deployment: LINE / Slack / Web / Cursor

### LINE Bot

```python
from flask import Flask, request
from linebot.v3.messaging import MessagingApi
from linebot.v3.webhooks import MessageEvent, TextMessageContent

@app.route("/webhook", methods=["POST"])
def webhook():
    events = parser.parse(request.get_data(as_text=True), signature)
    for event in events:
        if isinstance(event, MessageEvent):
            user_id = event.source.user_id
            question = event.message.text
            
            # Get user role from LINE user ID
            user_role = get_user_role(user_id)  # lookup จาก DB
            
            result = await handle_query(question, user_id, user_role)
            
            # Format สำหรับ LINE
            reply_text = f"{result['answer']}\n\n📚 แหล่งที่มา: {result['sources'][0]['file']}"
            line_bot_api.reply_message(event.reply_token, TextMessage(text=reply_text))
```

### Slack Bot

```python
from slack_bolt import App

@app.message()
def handle_message(message, say):
    question = message["text"]
    user_id = message["user"]
    user_role = get_slack_user_role(user_id)
    
    # Show typing indicator
    say("⏳ กำลังค้นหาข้อมูล...")
    
    result = handle_query(question, user_id, user_role)
    
    # Rich Slack formatting
    blocks = [
        {"type": "section", "text": {"type": "mrkdwn", "text": result["answer"]}},
        {"type": "divider"},
        {"type": "context", "elements": [
            {"type": "mrkdwn", 
             "text": f"📚 Source: {', '.join(s['file'] for s in result['sources'])} | ⚡ {result['latency_ms']}ms"}
        ]}
    ]
    say(blocks=blocks)
```

### Cursor (Dev MCP Server)

```python
# MCP Server สำหรับ Cursor IDE
# ให้ Dev ถาม business rules ขณะเขียนโค้ด

from mcp.server import Server

server = Server("sellsuki-knowledge")

@server.tool()
async def search_sellsuki_docs(query: str, context: str = "engineering") -> str:
    """ค้นหาข้อมูล Sellsuki docs สำหรับ developer"""
    result = await handle_query(
        question=query,
        user_id="dev-cursor",
        user_role="engineering"
    )
    return f"{result['answer']}\n\nSources: {result['sources']}"
```

---

## 🎯 สรุปสิ่งที่ได้จากระบบนี้

| สิ่งที่ได้ | รายละเอียด |
|---|---|
| **ความเร็ว** | cache hit < 5ms, cold query < 2s |
| **ความแม่นยำ** | ตอบจากเอกสารจริง อ้างอิงได้ |
| **ประหยัดค่าใช้จ่าย** | semantic cache ลด LLM calls 60-80% |
| **Access Control** | แต่ละ role เห็นข้อมูลที่อนุญาตเท่านั้น |
| **Always Fresh** | smart update — doc ใหม่ = answer ใหม่ทันที |
| **No duplicate work** | hash-based dedup ไม่ embed ซ้ำ |
| **Multi-channel** | LINE, Slack, Web, Cursor ใช้ backend เดียวกัน |
| **Observable** | LangSmith trace ทุก query |

---

*Built for Sellsuki — Powered by LangChain + ParadeDB + Dragonfly*
