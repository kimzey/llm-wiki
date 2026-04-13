# 🌾 Haystack Deep Dive — Phase 1: รู้จัก Haystack & แนวคิดพื้นฐาน

> **เป้าหมาย Phase 1:** เข้าใจว่า Haystack คืออะไร, สถาปัตยกรรมหลัก, และติดตั้งให้พร้อมใช้งาน

---

## 1. Haystack คืออะไร?

**Haystack** คือ Open-Source Framework สำหรับสร้าง **AI-powered Search & NLP Pipelines** พัฒนาโดย [deepset](https://www.deepset.ai/)

ออกแบบมาเพื่อ:
- สร้างระบบ **Question Answering (QA)** จากเอกสารจำนวนมาก
- สร้าง **RAG (Retrieval-Augmented Generation)** Pipeline
- สร้าง **Semantic Search** ที่เข้าใจความหมาย ไม่ใช่แค่คำตรงๆ
- รองรับ **LLM หลายตัว** เช่น OpenAI, Cohere, HuggingFace, Ollama ฯลฯ

### เปรียบเทียบ Haystack vs ทางเลือกอื่น

| Feature | Haystack | LangChain | LlamaIndex |
|---|---|---|---|
| Focus | Production-ready Pipelines | General LLM Apps | Data Indexing |
| Component Model | Modular & Typed | Flexible | Data-centric |
| Pipeline Validation | ✅ Schema-based | ❌ | ❌ |
| Async Support | ✅ | ✅ | ✅ |
| Learning Curve | ปานกลาง | ต่ำ | ปานกลาง |

---

## 2. สถาปัตยกรรมหลัก (Architecture)

Haystack ใช้ concept ของ **Pipeline** ที่ประกอบด้วย **Component** เชื่อมกันเป็นกราฟ (DAG)

```
[Documents] → [DocumentStore] → [Retriever] → [Ranker] → [Generator/Reader] → [Answer]
```

### องค์ประกอบหลัก 4 ส่วน

```
┌─────────────────────────────────────────────────────────┐
│                    HAYSTACK PIPELINE                    │
│                                                         │
│  ┌──────────┐    ┌───────────┐    ┌──────────────────┐  │
│  │Document  │    │ Retriever │    │    Generator     │  │
│  │  Store   │───▶│(BM25/Vec) │───▶│  (LLM/Reader)   │  │
│  └──────────┘    └───────────┘    └──────────────────┘  │
│       ▲                ▲                   │             │
│       │           ┌────┴────┐              ▼             │
│  [Indexing]       │  Query  │          [Answer]          │
│   Pipeline        └─────────┘                            │
└─────────────────────────────────────────────────────────┘
```

### 2.1 Component (คอมโพเนนต์)
หน่วยเล็กสุดใน Haystack แต่ละ Component รับ input → ประมวลผล → ส่ง output

ตัวอย่าง Component ที่ใช้บ่อย:

| ประเภท | ตัวอย่าง Component | หน้าที่ |
|---|---|---|
| **Converter** | `PyPDFToDocument`, `HTMLToDocument` | แปลงไฟล์เป็น Document |
| **Preprocessor** | `DocumentSplitter`, `DocumentCleaner` | ตัด/ทำความสะอาด Document |
| **Embedder** | `SentenceTransformersDocumentEmbedder` | สร้าง Vector |
| **Writer** | `DocumentWriter` | บันทึกลง DocumentStore |
| **Retriever** | `InMemoryBM25Retriever`, `QdrantEmbeddingRetriever` | ค้นหา Document |
| **Ranker** | `TransformersSimilarityRanker` | จัดลำดับผลลัพธ์ |
| **Generator** | `OpenAIGenerator`, `HuggingFaceLocalGenerator` | สร้างคำตอบ |
| **Builder** | `PromptBuilder` | สร้าง Prompt |

### 2.2 Pipeline (ไปป์ไลน์)
การเชื่อม Component หลายตัวเข้าด้วยกันเป็น workflow

```python
from haystack import Pipeline

pipeline = Pipeline()
pipeline.add_component("retriever", retriever)
pipeline.add_component("generator", generator)
pipeline.connect("retriever.documents", "generator.documents")
```

### 2.3 DocumentStore (ที่เก็บเอกสาร)
ฐานข้อมูลสำหรับเก็บ Document และ Vector

| DocumentStore | ประเภท | เหมาะกับ |
|---|---|---|
| `InMemoryDocumentStore` | In-memory | Development, ทดสอบ |
| `QdrantDocumentStore` | Vector DB | Production, scale ใหญ่ |
| `WeaviateDocumentStore` | Vector DB | Graph + Vector |
| `ElasticsearchDocumentStore` | Full-text + Vector | Enterprise |
| `PineconeDocumentStore` | Managed Vector DB | Cloud-native |
| `ChromaDocumentStore` | Local Vector DB | เริ่มต้นง่าย |

### 2.4 Document (เอกสาร)
โครงสร้างข้อมูลพื้นฐานใน Haystack

```python
from haystack.dataclasses import Document

doc = Document(
    content="เนื้อหาของเอกสาร...",
    meta={
        "source": "report.pdf",
        "author": "สมชาย",
        "date": "2024-01-01"
    },
    embedding=[0.1, 0.2, ...]  # Vector (ถ้ามี)
)
```

---

## 3. การติดตั้ง (Installation)

### 3.1 ติดตั้ง Haystack หลัก

```bash
pip install haystack-ai
```

### 3.2 ติดตั้ง Integration ตามที่ต้องการ

```bash
# OpenAI
pip install haystack-ai openai

# HuggingFace (Local Models)
pip install haystack-ai transformers torch

# Sentence Transformers (Embedding)
pip install sentence-transformers

# Vector Databases
pip install qdrant-client              # Qdrant
pip install weaviate-client            # Weaviate
pip install elasticsearch              # Elasticsearch
pip install chromadb                   # Chroma

# Document Converters
pip install pypdf                      # PDF
pip install python-docx                # Word
pip install trafilatura                # Web scraping

# ติดตั้งครบ (Development)
pip install haystack-ai[all]
```

### 3.3 ตรวจสอบการติดตั้ง

```python
import haystack
print(haystack.__version__)  # ควรได้ 2.x.x

# ตรวจสอบ Component ที่ใช้ได้
from haystack.components.generators import OpenAIGenerator
print("✅ OpenAI Generator พร้อมใช้งาน")
```

---

## 4. Hello World — RAG Pipeline แรก

ตัวอย่างง่ายที่สุด: ถาม-ตอบจากเอกสาร

```python
from haystack import Document, Pipeline
from haystack.components.retrievers.in_memory import InMemoryBM25Retriever
from haystack.components.generators import OpenAIGenerator
from haystack.components.builders import PromptBuilder
from haystack.document_stores.in_memory import InMemoryDocumentStore
import os

# 1. ตั้งค่า API Key
os.environ["OPENAI_API_KEY"] = "sk-..."

# 2. สร้าง DocumentStore และเพิ่มเอกสาร
document_store = InMemoryDocumentStore()
document_store.write_documents([
    Document(content="Python เป็นภาษาโปรแกรมที่ออกแบบโดย Guido van Rossum ในปี 1991"),
    Document(content="Python มีชื่อเสียงด้านความง่ายในการอ่านและเขียน"),
    Document(content="Python ใช้กันอย่างแพร่หลายใน Data Science และ AI"),
])

# 3. สร้าง Prompt Template
template = """
ตอบคำถามโดยอ้างอิงจากบริบทที่ให้มาเท่านั้น:

บริบท:
{% for doc in documents %}
  {{ doc.content }}
{% endfor %}

คำถาม: {{ question }}
คำตอบ:
"""

# 4. สร้าง Pipeline
pipeline = Pipeline()
pipeline.add_component("retriever", InMemoryBM25Retriever(document_store=document_store))
pipeline.add_component("prompt_builder", PromptBuilder(template=template))
pipeline.add_component("llm", OpenAIGenerator(model="gpt-4o-mini"))

# 5. เชื่อมต่อ Component
pipeline.connect("retriever", "prompt_builder.documents")
pipeline.connect("prompt_builder", "llm")

# 6. รัน Pipeline
result = pipeline.run({
    "retriever": {"query": "ใครสร้าง Python?"},
    "prompt_builder": {"question": "ใครสร้าง Python?"}
})

print(result["llm"]["replies"][0])
# Output: Python ออกแบบโดย Guido van Rossum ในปี 1991
```

---

## 5. โครงสร้าง Project ที่แนะนำ

```
my_haystack_project/
├── data/                    # เก็บเอกสาร source
│   ├── pdfs/
│   ├── htmls/
│   └── texts/
├── pipelines/               # Pipeline definitions
│   ├── indexing_pipeline.py
│   └── rag_pipeline.py
├── config/                  # Configuration
│   └── settings.py
├── utils/                   # Helper functions
├── tests/
└── main.py
```

---

## 6. สรุป Phase 1

| หัวข้อ | สิ่งที่ได้เรียนรู้ |
|---|---|
| Haystack คืออะไร | Framework สร้าง AI Search & RAG Pipeline |
| สถาปัตยกรรม | Component → Pipeline → DocumentStore |
| Document | โครงสร้างข้อมูลพื้นฐาน มี content + meta + embedding |
| การติดตั้ง | `pip install haystack-ai` + integrations |
| Hello World | RAG Pipeline พื้นฐานใช้ BM25 + OpenAI |

---

## 📚 Phase ถัดไป

- **Phase 2:** Document Processing — การโหลดและประมวลผลเอกสาร
- **Phase 3:** Embedding & Vector Search
- **Phase 4:** RAG Pipeline ขั้นสูง
- **Phase 5:** Custom Components & Production
