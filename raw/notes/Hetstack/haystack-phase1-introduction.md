# 🌾 Haystack Deep Dive — Phase 1: Introduction & Core Concepts

> **เป้าหมาย:** เข้าใจว่า Haystack คืออะไร ทำงานอย่างไร และทำไมถึงต้องใช้

---

## 1. Haystack คืออะไร?

**Haystack** คือ Open-source Framework สำหรับสร้างระบบ **NLP Pipeline** โดยเฉพาะงานด้าน:

- 🔍 **Question Answering (QA)** — ถามคำถาม ตอบจากเอกสาร
- 🤖 **RAG (Retrieval-Augmented Generation)** — ค้นหาข้อมูล แล้วให้ LLM สรุป/ตอบ
- 💬 **Conversational AI / Chatbot** — แชทบอทที่รู้เรื่องข้อมูลของคุณ
- 📄 **Document Search** — ค้นหาเอกสารอัจฉริยะ
- 🧠 **Agent Systems** — ระบบ AI ที่ตัดสินใจได้เอง

พัฒนาโดย **deepset** (บริษัทเยอรมัน) และใช้ Python เป็นหลัก

---

## 2. ทำไมต้องใช้ Haystack?

| ปัญหา | Haystack แก้ยังไง |
|-------|------------------|
| LLM ไม่รู้ข้อมูลใหม่ | ดึงข้อมูลจาก document store มาเสริม |
| ต้องต่อ component เยอะ | มี Pipeline สำเร็จรูป |
| Support หลาย LLM | รองรับ OpenAI, HuggingFace, Cohere ฯลฯ |
| Production-ready | มี Document Store, REST API, Monitoring |

---

## 3. Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                    Haystack Pipeline                 │
│                                                      │
│  [Document]  →  [Indexing Pipeline]  →  [Store]     │
│                                            ↓         │
│  [Query]     →  [Query Pipeline]    →  [Answer]     │
└─────────────────────────────────────────────────────┘
```

Haystack แบ่งเป็น 2 งานหลัก:

### 3.1 Indexing Pipeline (เตรียมข้อมูล)
```
Raw Documents → PreProcessor → Embedder → Document Store
```

### 3.2 Query Pipeline (ตอบคำถาม)
```
Question → Retriever → Reader/Generator → Answer
```

---

## 4. Component หลักของ Haystack

### 🗃️ Document Store
เป็น "ฐานข้อมูล" ที่เก็บเอกสาร รองรับ:

| Document Store | ประเภท | เหมาะกับ |
|----------------|--------|---------|
| `InMemoryDocumentStore` | In-memory | Development/Testing |
| `ElasticsearchDocumentStore` | Full-text search | Production, BM25 |
| `OpenSearchDocumentStore` | Full-text search | Production, cloud |
| `WeaviateDocumentStore` | Vector DB | Semantic search |
| `QdrantDocumentStore` | Vector DB | High performance |
| `PineconeDocumentStore` | Vector DB (cloud) | Managed cloud |
| `ChromaDocumentStore` | Vector DB | Local dev |
| `FAISSDocumentStore` | Vector index | Fast similarity search |

### 🔎 Retriever
ค้นหาเอกสารที่เกี่ยวข้องจาก Document Store:

| Retriever | วิธีค้นหา | ข้อดี |
|-----------|-----------|-------|
| `BM25Retriever` | Keyword matching | เร็ว ไม่ต้องการ embedding |
| `EmbeddingRetriever` | Vector similarity | เข้าใจความหมาย |
| `DensePassageRetriever` | Bi-encoder | เก่งมาก แต่หนัก |
| `SentenceTransformersRetriever` | Sentence embeddings | ดี balanced |

### 📖 Reader / Generator
ประมวลผลข้อมูลที่ค้นมาแล้วตอบคำถาม:

| Component | หน้าที่ |
|-----------|--------|
| `FARMReader` | Extractive QA (ตัดข้อความตอบ) |
| `TransformersReader` | Extractive QA via HuggingFace |
| `OpenAIGenerator` | Generative answer via GPT |
| `HuggingFaceGenerator` | Generative via HF models |

### ⚙️ Other Components

| Component | หน้าที่ |
|-----------|--------|
| `PreProcessor` | ตัดเอกสารเป็น chunk |
| `DocumentJoiner` | รวมผลจากหลาย Retriever |
| `PromptBuilder` | สร้าง prompt สำหรับ LLM |
| `OutputAdapter` | แปลง output format |

---

## 5. Haystack 1.x vs 2.x

> ⚠️ Haystack มี 2 versions ที่แตกต่างกันมาก

| | Haystack 1.x | Haystack 2.x (ปัจจุบัน) |
|-|--------------|------------------------|
| API | Pipeline-based (เก่า) | Component-based (ใหม่) |
| Install | `pip install farm-haystack` | `pip install haystack-ai` |
| Import | `from haystack` | `from haystack` |
| Flexibility | ปานกลาง | สูงมาก |
| Status | Maintenance mode | Active development |

**แนะนำให้ใช้ Haystack 2.x** ในเอกสารชุดนี้จะสอน **Haystack 2.x** ทั้งหมด

---

## 6. Installation

```bash
# ติดตั้ง Haystack 2.x หลัก
pip install haystack-ai

# ติดตั้งพร้อม integrations ที่ใช้บ่อย
pip install haystack-ai openai

# ถ้าใช้ HuggingFace models
pip install haystack-ai transformers torch sentence-transformers

# ถ้าใช้ Elasticsearch
pip install haystack-ai elasticsearch

# ถ้าใช้ Qdrant
pip install qdrant-haystack
```

---

## 7. Hello World — ตัวอย่างแรก

```python
from haystack import Document, Pipeline
from haystack.components.retrievers.in_memory import InMemoryBM25Retriever
from haystack.components.builders import PromptBuilder
from haystack.components.generators import OpenAIGenerator
from haystack.document_stores.in_memory import InMemoryDocumentStore

# 1. สร้าง Document Store และใส่เอกสาร
store = InMemoryDocumentStore()
store.write_documents([
    Document(content="Python ถูกสร้างโดย Guido van Rossum ในปี 1991"),
    Document(content="Python เป็นภาษาที่อ่านง่าย เหมาะกับผู้เริ่มต้น"),
    Document(content="Python ใช้ในด้าน Data Science, AI, Web Development"),
])

# 2. สร้าง Pipeline
pipeline = Pipeline()
pipeline.add_component("retriever", InMemoryBM25Retriever(document_store=store))
pipeline.add_component("prompt_builder", PromptBuilder(
    template="""ตอบคำถามจากบริบทต่อไปนี้:
    บริบท: {% for doc in documents %}{{ doc.content }}{% endfor %}
    คำถาม: {{ question }}
    คำตอบ:"""
))
pipeline.add_component("llm", OpenAIGenerator(model="gpt-4o-mini"))

# 3. เชื่อม component
pipeline.connect("retriever.documents", "prompt_builder.documents")
pipeline.connect("prompt_builder.prompt", "llm.prompt")

# 4. รัน
result = pipeline.run({
    "retriever": {"query": "Python ถูกสร้างโดยใคร?"},
    "prompt_builder": {"question": "Python ถูกสร้างโดยใคร?"}
})

print(result["llm"]["replies"][0])
# Output: "Python ถูกสร้างโดย Guido van Rossum ในปี 1991"
```

---

## 8. Document คืออะไร?

`Document` คือ class หลักที่เก็บข้อมูล:

```python
from haystack import Document

# Document พื้นฐาน
doc = Document(content="เนื้อหาของเอกสาร")

# Document พร้อม metadata
doc = Document(
    content="Python เป็นภาษา Programming",
    meta={
        "source": "wikipedia",
        "language": "th",
        "author": "John",
        "date": "2024-01-01",
    }
)

# Document แบบมี embedding (vector)
doc = Document(
    content="เนื้อหา",
    embedding=[0.1, 0.2, 0.3, ...]  # vector จาก embedding model
)

# เข้าถึง properties
print(doc.id)        # auto-generated unique ID
print(doc.content)   # เนื้อหา
print(doc.meta)      # metadata dict
print(doc.score)     # relevance score (หลัง retrieval)
```

---

## 9. Pipeline พื้นฐาน

```python
from haystack import Pipeline

# สร้าง pipeline
pipeline = Pipeline()

# เพิ่ม component
pipeline.add_component("component_name", ComponentInstance())

# เชื่อม output → input
pipeline.connect("component_a.output_name", "component_b.input_name")

# รัน pipeline
result = pipeline.run({
    "component_name": {"input_key": "value"}
})

# ดู output
print(result["component_name"]["output_key"])
```

---

## 10. สรุป Phase 1

```
✅ Haystack = Framework สร้าง NLP/AI Pipeline
✅ ใช้งาน RAG, QA, Chatbot, Agent
✅ มี Document Store, Retriever, Reader/Generator
✅ ใช้ Haystack 2.x (pip install haystack-ai)
✅ Pipeline = ต่อ Component เป็น chain
✅ Document = unit ของข้อมูลใน Haystack
```

---

**➡️ ต่อไป: Phase 2 — Document Processing & Indexing Pipeline**
