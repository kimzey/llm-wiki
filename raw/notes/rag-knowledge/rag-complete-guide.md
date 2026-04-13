# RAG (Retrieval-Augmented Generation) — คู่มือฉบับสมบูรณ์

> รวบรวมทุกเรื่องเกี่ยวกับ RAG ตั้งแต่พื้นฐานจนถึง Production Deployment

---

## สารบัญ

1. [RAG คืออะไร](#1-rag-คืออะไร)
2. [RAG Pipeline ทั้ง 8 ขั้นตอน](#2-rag-pipeline-ทั้ง-8-ขั้นตอน)
3. [Chunking — วิธีตัดข้อมูล](#3-chunking--วิธีตัดข้อมูล)
4. [Embedding — แปลงข้อความเป็น Vector](#4-embedding--แปลงข้อความเป็น-vector)
5. [Vector Store — เก็บข้อมูล](#5-vector-store--เก็บข้อมูล)
6. [Retrieval — วิธีค้นหา](#6-retrieval--วิธีค้นหา)
7. [Framework เปรียบเทียบ](#7-framework-เปรียบเทียบ)
8. [LlamaIndex — รายละเอียด](#8-llamaindex--รายละเอียด)
9. [Haystack — รายละเอียด](#9-haystack--รายละเอียด)
10. [LangChain + LangServe](#10-langchain--langserve)
11. [วิธีทำ RAG ให้คนภายนอกใช้](#11-วิธีทำ-rag-ให้คนภายนอกใช้)
12. [Deployment — Deploy ได้ที่ไหนบ้าง](#12-deployment--deploy-ได้ที่ไหนบ้าง)
13. [Evaluation — วัดคุณภาพ](#13-evaluation--วัดคุณภาพ)
14. [แนะนำตามสถานการณ์](#14-แนะนำตามสถานการณ์)

---

## 1. RAG คืออะไร

RAG (Retrieval-Augmented Generation) คือเทคนิคที่ทำให้ LLM ตอบคำถามจาก **ข้อมูลของเรา** ได้ โดยการค้นหาข้อมูลที่เกี่ยวข้องก่อน แล้วส่งให้ LLM สร้างคำตอบ

```
User ถามคำถาม
    ↓
ค้นหาข้อมูลที่เกี่ยวข้องจาก database ของเรา
    ↓
ยัดข้อมูลเข้า prompt
    ↓
LLM สร้างคำตอบจากข้อมูลนั้น
    ↓
User ได้คำตอบที่ถูกต้อง + มีแหล่งอ้างอิง
```

### ทำไมต้อง RAG?

- LLM ไม่รู้ข้อมูลภายในองค์กร (นโยบาย HR, เอกสารบริษัท)
- LLM มี knowledge cutoff ข้อมูลไม่ update
- ลด hallucination เพราะตอบจากข้อมูลจริง
- มีแหล่งอ้างอิงให้ตรวจสอบได้

---

## 2. RAG Pipeline ทั้ง 8 ขั้นตอน

```
เอกสาร → Load → Chunk → Embed → Store → Retrieve → Augment → Generate → คำตอบ
```

| ขั้นตอน      | ทำอะไร                   | ตัวอย่าง                     |
| ------------ | ------------------------ | ---------------------------- |
| **Load**     | โหลดข้อมูลเข้ามา         | PDF, Word, Notion, Database  |
| **Chunk**    | ตัดเป็นชิ้นเล็กๆ         | ตัดตามประโยค, ตามความหมาย    |
| **Embed**    | แปลงเป็นตัวเลข (vector)  | OpenAI embedding, BGE-M3     |
| **Store**    | เก็บใน vector database   | ChromaDB, Pinecone, Qdrant   |
| **Retrieve** | ค้นหาข้อมูลที่เกี่ยวข้อง | Vector search, Hybrid search |
| **Augment**  | ประกอบ prompt            | ใส่ context + คำถาม          |
| **Generate** | สร้างคำตอบ               | GPT-4, Claude, Gemini        |
| **Evaluate** | วัดคุณภาพ                | RAGAS framework              |

### ตัวอย่าง RAG แบบง่ายสุด (ไม่ใช้ Framework)

```python
import chromadb
from openai import OpenAI

client = OpenAI()
db = chromadb.Client()
collection = db.create_collection("docs")

# 1-4: Chunk + Embed + Store
collection.add(documents=["ลาป่วยได้ 30 วัน...", "WFH ได้ 2 วัน..."], ids=["doc1", "doc2"])

# 5: Retrieve
results = collection.query(query_texts=["ลาป่วยกี่วัน"], n_results=3)

# 6-7: Augment + Generate
context = "\n".join(results["documents"][0])
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": f"ตอบจาก context นี้: {context}"},
        {"role": "user", "content": "ลาป่วยกี่วัน?"}
    ]
)
print(response.choices[0].message.content)
```

---

## 3. Chunking — วิธีตัดข้อมูล

> Chunking เป็นขั้นตอนที่สำคัญที่สุด ตัดดีหรือไม่ดี กระทบคุณภาพคำตอบทั้งหมด

### 3.1 Fixed Size — ตัดตามขนาด

```
chunk 1: "AAAAA"    (500 chars)
chunk 2: "AABBB"    (500 chars, overlap 100)
chunk 3: "BBBBB"    (500 chars, overlap 100)
```

- ง่ายสุด เร็วสุด
- อาจตัดกลางประโยค กลางความหมาย
- เหมาะกับ prototype

### 3.2 Sentence — ตัดตามประโยค

```
chunk 1: "บริษัทก่อตั้งปี 2010. มีพนักงาน 500 คน. สำนักงานอยู่กรุงเทพ."
chunk 2: "สวัสดิการมีประกันสุขภาพ. ค่ารักษาพยาบาลปีละ 100,000."
```

- ไม่ตัดกลางประโยค
- **นิยมที่สุดสำหรับงานทั่วไป**

### 3.3 Recursive — ตัดหลายระดับ

```
ลองตัดด้วย "\n\n" (paragraph) ก่อน
  → ถ้ายังใหญ่เกิน ตัดด้วย "\n" (บรรทัด)
    → ถ้ายังใหญ่เกิน ตัดด้วย ". " (ประโยค)
      → ถ้ายังใหญ่เกิน ตัดด้วย " " (คำ)
```

- เป็น default ของ LangChain (`RecursiveCharacterTextSplitter`)
- พยายามรักษาโครงสร้างเดิมให้มากที่สุด

### 3.4 Semantic — ตัดตามความหมาย

```
1. แปลงแต่ละประโยคเป็น embedding
2. วัดความคล้ายระหว่างประโยคที่ติดกัน
3. ตรงไหนความคล้ายตกฮวบ = รอยต่อของหัวข้อ

ประโยค 1: "ลาป่วยได้ 30 วัน"        ─┐
ประโยค 2: "ต้องมีใบรับรองแพทย์"      ─┤ → chunk A (เรื่องลาป่วย)
ประโยค 3: "ลาเกิน 3 วันต้องแจ้ง HR"  ─┘
                                         ← ความคล้ายตก
ประโยค 4: "ค่ารักษาพยาบาล 100,000"   ─┐
ประโยค 5: "ครอบคลุมทันตกรรม"         ─┤ → chunk B (เรื่องสวัสดิการ)
ประโยค 6: "เบิกค่าแว่นได้ 3,000"     ─┘
```

- chunk ไม่เท่ากัน แต่แต่ละ chunk พูดเรื่องเดียวกัน
- แม่นที่สุดแต่ช้าที่สุด

### 3.5 Hierarchical (Parent-Child) — ตัดหลายขนาด

```
Parent chunk (2048 chars):
  "ทั้งหมดเกี่ยวกับนโยบายการลา..."
  └─ Child chunk 1 (256 chars): "ลาป่วย 30 วัน..."
  └─ Child chunk 2 (256 chars): "ลาพักร้อน 10 วัน..."
  └─ Child chunk 3 (256 chars): "ลาคลอด 90 วัน..."

ค้นหาด้วย child (แม่น) → ส่ง parent ให้ LLM (context เยอะ)
```

### 3.6 Structure-Based — ตัดตามโครงสร้าง

| โครงสร้าง | ตัดตาม |
|-----------|--------|
| Markdown | `# header` |
| HTML | `<section>`, `<article>` |
| Code | function, class |
| PDF | page, section |
| Table | แต่ละแถวหรือกลุ่มแถว |

### 3.7 Agentic Chunking — ให้ LLM ตัดให้

- ส่งเอกสารให้ LLM แล้วบอก "แบ่งเอกสารนี้เป็น chunk ที่แต่ละ chunk พูดเรื่องเดียวกัน"
- แม่นที่สุด แต่แพงที่สุด ช้าที่สุด

### สรุป Chunking

```
ง่าย/เร็ว ←──────────────────────────→ แม่น/ช้า

Fixed → Recursive → Sentence → Structure → Semantic → Agentic
  ↑                    ↑                      ↑
  prototype         ส่วนใหญ่ใช้             คุณภาพสูง
```

---

## 4. Embedding — แปลงข้อความเป็น Vector

แปลง chunk เป็น vector (array of numbers) เพื่อเปรียบเทียบความหมาย

```
"ลาป่วยได้ 30 วัน"   → [0.021, -0.043, 0.018, ..., 0.033]   # 1536 ตัวเลข
"sick leave 30 days" → [0.019, -0.041, 0.020, ..., 0.031]   # ใกล้กันมาก!
"วันหยุดนักขัตฤกษ์"   → [0.087, 0.012, -0.055, ..., -0.022]  # ไกลออกไป
```

### Embedding Models

| Model                         | มิติ | ราคา    | หมายเหตุ         |
| ----------------------------- | ---- | ------- | ---------------- |
| OpenAI text-embedding-3-small | 1536 | ถูกมาก  | นิยมสุด          |
| OpenAI text-embedding-3-large | 3072 | แพงกว่า | แม่นกว่าเล็กน้อย |
| Cohere embed-v3               | 1024 | ปานกลาง | รองรับหลายภาษาดี |
| Google text-embedding-004     | 768  | ถูก     | ใช้กับ Vertex AI |
| BGE-M3 (open source)          | 1024 | ฟรี     | self-host ได้    |
| multilingual-e5-large         | 1024 | ฟรี     | ภาษาไทยดี        |

---

## 5. Vector Store — เก็บข้อมูล

| ชื่อ | ประเภท | เหมาะกับ |
|------|--------|----------|
| ChromaDB | Open source, local | prototype, เริ่มต้น |
| Pinecone | Managed cloud | production, ไม่อยากดูแล |
| Qdrant | Open source | production, self-host |
| Weaviate | Open source | hybrid search |
| Milvus | Open source | ข้อมูลเยอะมากๆ |
| pgvector | Postgres extension | มี Postgres อยู่แล้ว |
| FAISS | Library (Meta) | ใน memory, เร็วมาก |

```python
# ตัวอย่าง: เก็บ vector ใน ChromaDB
collection.add(
    ids=["chunk_001", "chunk_002"],
    embeddings=[[0.021, -0.043, ...], [0.087, 0.012, ...]],
    documents=["ลาป่วยได้ 30 วัน...", "วันหยุดนักขัตฤกษ์..."],
    metadatas=[
        {"source": "hr_policy.pdf", "category": "leave"},
        {"source": "hr_policy.pdf", "category": "holiday"}
    ]
)
```

---

## 6. Retrieval — วิธีค้นหา

### 6.1 Vector Search (Semantic Search)

```
query: "หยุดกี่วันตอนปีใหม่"
  → embed query → หา vector ที่ใกล้ที่สุด
  → ผลลัพธ์: "วันหยุดนักขัตฤกษ์ปีใหม่ 31 ธ.ค. - 1 ม.ค."

ข้อดี:  เข้าใจความหมาย "หยุด" ≈ "ลา" ≈ "วันหยุด"
ข้อเสีย: อาจพลาดคำเฉพาะทาง, ชื่อเฉพาะ
```

### 6.2 Keyword Search (BM25)

```
query: "มาตรา 41 พ.ร.บ.คุ้มครองแรงงาน"
  → ค้นหาคำตรงๆ

ข้อดี:  แม่นกับชื่อเฉพาะ, รหัส, ตัวเลข
ข้อเสีย: ไม่เข้าใจ synonym
```

### 6.3 Hybrid Search (แนะนำสำหรับ Production)

```
query → Vector Search → ผลลัพธ์ A (10 ชิ้น)
query → BM25 Search   → ผลลัพธ์ B (10 ชิ้น)
รวมด้วย Reciprocal Rank Fusion (RRF) → ผลลัพธ์สุดท้าย
```

### 6.4 Reranking

```
ขั้นที่ 1: ดึงมา 20 chunks (เร็ว แต่หยาบ)
ขั้นที่ 2: cross-encoder ให้คะแนนทีละคู่ (ช้า แต่แม่น)
ขั้นที่ 3: เอา top 3 ส่งให้ LLM
```

### 6.5 Multi-Query Retrieval

```
คำถามเดิม: "บริษัทดูแลสุขภาพพนักงานยังไง"

LLM สร้างคำถามเพิ่ม:
  → "สวัสดิการสุขภาพพนักงาน"
  → "ประกันสุขภาพกลุ่ม"
  → "ตรวจสุขภาพประจำปี"

ค้นทั้ง 4 คำถาม → รวมผลลัพธ์ → ครอบคลุมมากขึ้น
```

### 6.6 Metadata Filtering

```python
results = vectorstore.query(
    query="ลาป่วย",
    filter={"category": "HR", "year": {"$gte": 2024}},
    top_k=5
)
```

### สรุป Retrieval

```
แม่นน้อย ←──────────────────────────→ แม่นมาก

Vector only → + BM25 (Hybrid) → + Reranking → + Multi-Query
    ↑              ↑                  ↑
    เริ่มต้น       production         คุณภาพสูงสุด
```

---

## 7. Framework เปรียบเทียบ

### ใครเป็นใคร

| Framework | เป็น RAG โดยตรง? | จุดเด่น |
|-----------|------------------|---------|
| **LangChain** | ใช่ | ทำได้ทุกอย่าง ecosystem ใหญ่ |
| **LlamaIndex** | ใช่ เกิดมาเพื่อสิ่งนี้ | indexing & retrieval ดีมาก |
| **Haystack** | ใช่ | production pipeline, enterprise |
| **Google ADK** | ไม่ตรงๆ | เน้น agent orchestration |

### เปรียบเทียบ

| ด้าน | LangChain | LlamaIndex | Haystack |
|------|-----------|------------|----------|
| ปรัชญา | ทำได้ทุกอย่าง | ทำให้ง่ายที่สุด | ควบคุมทุก step |
| Code เริ่มต้น | ~20 บรรทัด | ~5 บรรทัด | ~30-50 บรรทัด |
| ความยืดหยุ่น | สูง | ดี | สูงมาก |
| Production | ดี | ต้องจัดการเอง | ออกแบบมาเพื่อ production |
| Debugging | ปานกลาง | ยากกว่า (abstraction เยอะ) | ง่ายกว่า (เห็น flow ชัด) |
| Community | ใหญ่สุด | ใหญ่ | เล็กกว่า แต่ enterprise เยอะ |
| ได้ Server | ✅ LangServe | ต่อ FastAPI เอง | ✅ Hayhooks |
| TypeScript | ✅ LangChain.js | ✅ (feature น้อยกว่า) | ❌ Python only |

---

## 8. LlamaIndex — รายละเอียด

### สถาปัตยกรรม: 5 Stages

**Loading → Indexing → Storing → Querying → Evaluation**

### Loading — โหลดข้อมูล

```python
from llama_index.core import SimpleDirectoryReader

# โหลดทุกไฟล์ในโฟลเดอร์ (PDF, TXT, DOCX, etc.)
documents = SimpleDirectoryReader("./company_data").load_data()
```

### Indexing — สร้าง Index

```python
from llama_index.core import VectorStoreIndex, Settings
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding

Settings.llm = OpenAI(model="gpt-4")
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")
Settings.chunk_size = 512
Settings.chunk_overlap = 50

# chunk + embed ให้อัตโนมัติ
index = VectorStoreIndex.from_documents(documents)
```

**Index Types:**
- `VectorStoreIndex` — ใช้บ่อยสุด
- `SummaryIndex` — สร้าง summary
- `TreeIndex` — tree structure สำหรับเอกสารยาว
- `KeywordTableIndex` — ใช้ keyword ค้นหา

### Storing — เก็บ Index

```python
# บันทึก
index.storage_context.persist(persist_dir="./storage")

# โหลดกลับ (ไม่ต้อง embed ใหม่)
from llama_index.core import StorageContext, load_index_from_storage
storage_context = StorageContext.from_defaults(persist_dir="./storage")
index = load_index_from_storage(storage_context)
```

### Querying — ถามคำถาม

```python
# แบบพื้นฐาน
query_engine = index.as_query_engine(similarity_top_k=3)
response = query_engine.query("นโยบายลาป่วยเป็นยังไง?")
print(response)

# ดู source
for node in response.source_nodes:
    print(f"จาก: {node.metadata['file_name']}, score: {node.score:.3f}")
```

```python
# แบบ Chat (จำบทสนทนาได้)
chat_engine = index.as_chat_engine(chat_mode="condense_plus_context")
response1 = chat_engine.chat("บริษัทมีนโยบาย WFH ไหม?")
response2 = chat_engine.chat("แล้วต้องขออนุมัติล่วงหน้ากี่วัน?")
# มันรู้ว่าหมายถึงเรื่อง WFH
```

```python
# แบบ Sub-Question (ถามซับซ้อน แตกคำถามเอง)
from llama_index.core.query_engine import SubQuestionQueryEngine
from llama_index.core.tools import QueryEngineTool

hr_tool = QueryEngineTool.from_defaults(
    query_engine=hr_index.as_query_engine(),
    description="ข้อมูลนโยบาย HR"
)
finance_tool = QueryEngineTool.from_defaults(
    query_engine=finance_index.as_query_engine(),
    description="ข้อมูลการเงินบริษัท"
)

query_engine = SubQuestionQueryEngine.from_defaults(
    query_engine_tools=[hr_tool, finance_tool]
)

response = query_engine.query(
    "งบสวัสดิการปีนี้เทียบปีที่แล้วเป็นยังไง?"
)
# แตกเป็น sub-questions → ถามแต่ละ tool → รวมคำตอบ
```

### Chunking ใน LlamaIndex

```python
from llama_index.core.node_parser import (
    SentenceSplitter,              # ตัดตามประโยค (default)
    TokenTextSplitter,             # ตัดตามจำนวน token
    SemanticSplitterNodeParser,    # ตัดตามความหมาย
    HierarchicalNodeParser,        # ตัดเป็นลำดับชั้น
    MarkdownNodeParser,            # ตัดตาม markdown header
    HTMLNodeParser,                # ตัดตาม HTML tag
    CodeSplitter,                  # ตัดตามโครงสร้าง code
    SentenceWindowNodeParser       # ตัดเป็นประโยค + context
)

# Semantic chunking
splitter = SemanticSplitterNodeParser(
    embed_model=OpenAIEmbedding(),
    breakpoint_percentile_threshold=95,
)

# Hierarchical chunking
splitter = HierarchicalNodeParser.from_defaults(
    chunk_sizes=[2048, 512, 128]
)
```

---

## 9. Haystack — รายละเอียด

### สถาปัตยกรรม: Component + Pipeline

ทุกอย่างคือ Component ต่อกันเป็น Pipeline

### Indexing Pipeline

```python
from haystack import Pipeline
from haystack.components.converters import PyPDFToDocument
from haystack.components.preprocessors import DocumentCleaner, DocumentSplitter
from haystack.components.embedders import OpenAIDocumentEmbedder
from haystack.components.writers import DocumentWriter
from haystack.document_stores.in_memory import InMemoryDocumentStore

document_store = InMemoryDocumentStore()

indexing = Pipeline()
indexing.add_component("converter", PyPDFToDocument())
indexing.add_component("cleaner", DocumentCleaner())
indexing.add_component("splitter", DocumentSplitter(
    split_by="sentence", split_length=5, split_overlap=1
))
indexing.add_component("embedder", OpenAIDocumentEmbedder())
indexing.add_component("writer", DocumentWriter(document_store=document_store))

indexing.connect("converter", "cleaner")
indexing.connect("cleaner", "splitter")
indexing.connect("splitter", "embedder")
indexing.connect("embedder", "writer")

indexing.run({"converter": {"sources": ["hr_policy.pdf"]}})
```

### Query Pipeline

```python
from haystack.components.embedders import OpenAITextEmbedder
from haystack.components.retrievers.in_memory import InMemoryEmbeddingRetriever
from haystack.components.builders import PromptBuilder
from haystack.components.generators import OpenAIGenerator

template = """
ตอบจากข้อมูลนี้:
{% for doc in documents %}
  {{ doc.content }}
{% endfor %}

คำถาม: {{ question }}
"""

query = Pipeline()
query.add_component("embedder", OpenAITextEmbedder())
query.add_component("retriever", InMemoryEmbeddingRetriever(
    document_store=document_store, top_k=3
))
query.add_component("prompt", PromptBuilder(template=template))
query.add_component("llm", OpenAIGenerator(model="gpt-4"))

query.connect("embedder.embedding", "retriever.query_embedding")
query.connect("retriever.documents", "prompt.documents")
query.connect("prompt", "llm")

result = query.run({
    "embedder": {"text": "ลาป่วยได้กี่วัน?"},
    "prompt": {"question": "ลาป่วยได้กี่วัน?"}
})
```

### Hybrid Search + Reranking

```python
from haystack.components.retrievers.in_memory import (
    InMemoryEmbeddingRetriever,
    InMemoryBM25Retriever
)
from haystack.components.joiners import DocumentJoiner
from haystack.components.rankers import TransformersSimilarityRanker

hybrid = Pipeline()

# Vector + BM25
hybrid.add_component("embedder", OpenAITextEmbedder())
hybrid.add_component("vector_retriever", InMemoryEmbeddingRetriever(document_store=document_store, top_k=10))
hybrid.add_component("bm25_retriever", InMemoryBM25Retriever(document_store=document_store, top_k=10))

# รวมผลลัพธ์
hybrid.add_component("joiner", DocumentJoiner(join_mode="reciprocal_rank_fusion"))

# Rerank
hybrid.add_component("ranker", TransformersSimilarityRanker(
    model="cross-encoder/ms-marco-MiniLM-L-6-v2", top_k=3
))

hybrid.connect("embedder", "vector_retriever")
hybrid.connect("vector_retriever", "joiner")
hybrid.connect("bm25_retriever", "joiner")
hybrid.connect("joiner", "ranker")
```

### Custom Component

```python
from haystack import component, Document

@component
class ThaiSentenceSplitter:
    @component.output_types(documents=list[Document])
    def run(self, documents: list[Document]):
        from pythainlp.tokenize import sent_tokenize
        result = []
        for doc in documents:
            sentences = sent_tokenize(doc.content)
            for i in range(0, len(sentences), 3):
                chunk = " ".join(sentences[i:i+3])
                result.append(Document(content=chunk, meta={**doc.meta}))
        return {"documents": result}
```

---

## 10. LangChain + LangServe

### RAG Chain

```python
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import Chroma
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

vectorstore = Chroma(persist_directory="./db", embedding_function=OpenAIEmbeddings())
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

prompt = ChatPromptTemplate.from_template("""
ตอบจากข้อมูลนี้: {context}
คำถาม: {question}
""")

chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt
    | ChatOpenAI(model="gpt-4o-mini")
    | StrOutputParser()
)
```

### LangServe — ได้ API อัตโนมัติ

```python
from fastapi import FastAPI
from langserve import add_routes

app = FastAPI(title="HR Bot API")
add_routes(app, chain, path="/ask")

# uvicorn server:app --port 8000
# API: POST http://localhost:8000/ask/invoke
# Playground: http://localhost:8000/ask/playground
# Docs: http://localhost:8000/docs
```

### LangChain.js (TypeScript)

```typescript
import { ChatOpenAI } from "@langchain/openai";
import { OpenAIEmbeddings } from "@langchain/openai";
import { Chroma } from "@langchain/community/vectorstores/chroma";
import { RecursiveCharacterTextSplitter } from "langchain/text_splitter";
import { createRetrievalChain } from "langchain/chains/retrieval";

const splitter = new RecursiveCharacterTextSplitter({
  chunkSize: 500,
  chunkOverlap: 50,
});

const docs = await new TextLoader("hr_policy.txt").load();
const chunks = await splitter.splitDocuments(docs);
const vectorStore = await Chroma.fromDocuments(chunks, new OpenAIEmbeddings());

const retriever = vectorStore.asRetriever({ k: 3 });
const chain = await createRetrievalChain({ retriever, combineDocsChain });
const result = await chain.invoke({ input: "ลาป่วยได้กี่วัน?" });
```

---

## 11. วิธีทำ RAG ให้คนภายนอกใช้

### แนวทาง 1: No-Code — ไม่ต้องเขียน Code

| Platform | วิธีทำ | ราคา | ได้อะไร |
|----------|--------|------|---------|
| **OpenAI GPTs** | อัพโหลดไฟล์ → สร้าง GPT | จ่ายตาม token | Share link / API |
| **Chatbase** | อัพโหลด → ได้ chatbot | $19/เดือน+ | Embed widget |
| **CustomGPT** | อัพโหลด → ฝังเว็บ | $49/เดือน+ | Embed widget |
| **Vertex AI Search** | อัพโหลด → สร้าง App | จ่ายตาม query | Widget / API |
| **AWS Bedrock KB** | S3 → Knowledge Base | จ่ายตาม query | API endpoint |
| **Dante AI** | อัพโหลด → embed | $9/เดือน+ | Embed widget |

### แนวทาง 2: Low-Code — เขียนนิดหน่อย

| Platform | วิธีทำ | ข้อดี |
|----------|--------|-------|
| **Dify** | Docker → ลาก วาง → API + UI | ฟรี, ครบจบ, UI สวย |
| **Flowise** | npm install → ลาก วาง → API | ฟรี, เห็น pipeline |
| **Langflow** | pip install → ลาก วาง → API | ฟรี, LangChain ecosystem |
| **Streamlit** | Python 20 บรรทัด → เว็บ chat | ง่ายสุด, deploy ฟรี |

**Dify ตัวอย่าง:**

```bash
docker compose up -d
# เปิด http://localhost/install
# สร้าง knowledge base → อัพโหลดเอกสาร → สร้าง chatbot
# ได้: Share link + API + Embed widget + Analytics
```

**Streamlit ตัวอย่าง:**

```python
import streamlit as st
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader

st.title("ถามข้อมูลบริษัท")

@st.cache_resource
def load_index():
    return VectorStoreIndex.from_documents(
        SimpleDirectoryReader("./data").load_data()
    )

index = load_index()
chat_engine = index.as_chat_engine()

if question := st.chat_input("ถามคำถาม"):
    st.chat_message("user").write(question)
    response = chat_engine.chat(question)
    st.chat_message("assistant").write(str(response))
```

### แนวทาง 3: Framework — LangChain, LlamaIndex, Haystack

(ดูรายละเอียดในหัวข้อ 8-10)

สิ่งที่ Framework ให้: Document loading, Chunking, Embedding, Vector storage, Retrieval, Prompt construction, LLM calling, Source tracking

สิ่งที่ต้องทำเอง: API server, UI, Auth / rate limiting, Deployment, Monitoring

### แนวทาง 4: เขียนเองทั้งหมด

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import chromadb
from openai import OpenAI

app = FastAPI()
app.add_middleware(CORSMiddleware, allow_origins=["*"])

ai = OpenAI()
db = chromadb.PersistentClient("./vectordb")
col = db.get_collection("company_docs")

@app.post("/ask")
async def ask(body: dict):
    q = body["question"]
    results = col.query(query_texts=[q], n_results=3)
    context = "\n---\n".join(results["documents"][0])

    resp = ai.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": f"ตอบจากข้อมูลนี้เท่านั้น:\n{context}"},
            {"role": "user", "content": q}
        ]
    )
    return {
        "answer": resp.choices[0].message.content,
        "sources": results["metadatas"][0]
    }
```

### เปรียบเทียบทุกแนวทาง

| | ความง่าย | ยืดหยุ่น | ราคา | เวลา | เหมาะกับ |
|---|---|---|---|---|---|
| **No-Code** | ★★★★★ | ★☆☆☆☆ | $9-49/เดือน | 30 นาที | ลองเร็ว, เว็บบริษัท |
| **Low-Code** | ★★★★☆ | ★★★☆☆ | ฟรี (host เอง) | 2-3 ชม. | startup, ทีมเล็ก |
| **Framework** | ★★★☆☆ | ★★★★☆ | ฟรี (host เอง) | 1-3 วัน | production |
| **เขียนเอง** | ★★☆☆☆ | ★★★★★ | ฟรี (host เอง) | 3-7 วัน | เรียนรู้, ควบคุมเต็มที่ |

---

## 12. Deployment — Deploy ได้ที่ไหนบ้าง

### Official Platform

| Platform | วิธี | ได้อะไร |
|----------|------|---------|
| **LangSmith** | `langchain deploy` | API + Playground + Monitoring |

### Cloud Providers

| Platform | วิธี | เหมาะกับ |
|----------|------|----------|
| **Google Cloud Run** | `gcloud run deploy` | production, auto-scale, จ่ายตามใช้ |
| **AWS ECS / Fargate** | Docker container | enterprise, auto-scaling |
| **AWS Lambda** | Serverless | traffic น้อย |
| **Azure Container Apps** | Docker container | Microsoft ecosystem |

### PaaS (ง่าย ไม่ต้องจัดการ Server)

| Platform | วิธี | ราคา |
|----------|------|------|
| **Railway** | `railway up` | $5/เดือน+ |
| **Render** | push git → auto deploy | มี free tier |
| **Fly.io** | `fly deploy` | $0+ เลือก region ได้ |
| **Heroku** | `git push heroku main` | $5/เดือน+ |

### Serverless

| Platform | ภาษา | เหมาะกับ |
|----------|------|----------|
| **Vercel** | TypeScript only | frontend + API รวม |
| **Modal** | Python | ML workloads, scale to zero |

### Self-Hosted (Docker)

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["uvicorn", "server:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
docker build -t my-rag .
docker run -p 8000:8000 my-rag
```

### แนะนำ Deployment

| สถานการณ์ | แนะนำ |
|-----------|-------|
| เพิ่งเริ่ม ลองเล่น | Railway หรือ Render |
| Production ไทย | Cloud Run (Singapore) หรือ AWS ECS (Singapore) |
| งบน้อย traffic ไม่แน่นอน | Modal หรือ Cloud Run (จ่ายตามใช้) |
| TypeScript | Vercel + Next.js |

---

## 13. Evaluation — วัดคุณภาพ

### RAGAS Framework

| ตัววัด | วัดอะไร | ตัวอย่าง |
|--------|---------|----------|
| **Faithfulness** | คำตอบตรงกับ source ไหม? | "ลาได้ 30 วัน" ✅ / "ลาได้ 45 วัน" ❌ |
| **Answer Relevancy** | คำตอบตรงคำถามไหม? | ถามลาป่วย ตอบเรื่องลา ✅ / ตอบเรื่องที่จอดรถ ❌ |
| **Context Precision** | ดึง context ถูกต้องไหม? | ค้นได้ chunk ที่เกี่ยวข้อง ✅ |
| **Context Recall** | ดึง context มาครบไหม? | มี 3 chunk เกี่ยวข้อง ค้นได้ 3 ✅ |

```python
from llama_index.core.evaluation import FaithfulnessEvaluator, RelevancyEvaluator

faith_eval = FaithfulnessEvaluator()
result = faith_eval.evaluate_response(response=response)
print(f"Faithful: {result.passing}")  # True/False
```

---

## 14. แนะนำตามสถานการณ์

### "อยากลองก่อน ดูว่า RAG เวิร์คไหม"

→ **OpenAI Assistants** หรือ **Chatbase**: อัพโหลดไฟล์ ได้ chatbot ใน 30 นาที

### "ทำให้ลูกค้า/พนักงานใช้จริง แต่ไม่อยากเขียน code"

→ **Dify** self-host: ได้ UI สวย + API + Analytics ครบ

### "ทำ production จริงจัง ต้อง custom ได้"

→ **LangChain + LangServe** หรือ **LlamaIndex + FastAPI**

### "ต้องควบคุมทุกอย่าง เข้าใจทุก line"

→ **เขียนเอง**: FastAPI + ChromaDB + OpenAI

### "องค์กรใหญ่ มี compliance"

→ **AWS Bedrock Knowledge Bases** หรือ **Azure AI Search**

### เริ่มต้นแนะนำ

1. Sentence chunking
2. OpenAI text-embedding-3-small
3. ChromaDB
4. Vector search
5. GPT-4o-mini

ทำได้ใน 30 นาที แล้วค่อยปรับแต่ละขั้นให้ดีขึ้น ส่วนใหญ่คุณภาพจะดีขึ้นมากที่สุดจากการ **ปรับ chunking** กับ **เพิ่ม hybrid search + reranking**

---

> **สร้างจากการสนทนากับ Claude — มีนาคม 2026**
