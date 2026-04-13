# LlamaIndex Deep Dive — Phase 1: Introduction & Core Concepts

> **ซีรีส์นี้แบ่งเป็น 4 Phase:**
> - **Phase 1** → Introduction & Core Concepts *(ไฟล์นี้)*
> - **Phase 2** → Data Ingestion & Indexing
> - **Phase 3** → Querying, Retrieval & RAG Pipeline
> - **Phase 4** → Advanced Features, Agents & Production

---

## 1. LlamaIndex คืออะไร?

**LlamaIndex** (เดิมชื่อ GPT Index) คือ **Data Framework สำหรับ LLM Applications** ที่ช่วยให้เราสามารถเชื่อมต่อข้อมูลของเราเข้ากับ Large Language Models (LLMs) ได้อย่างมีประสิทธิภาพ

### ปัญหาที่ LlamaIndex แก้ไข

LLMs อย่าง GPT-4 หรือ Claude มี **knowledge cutoff** และไม่รู้จักข้อมูลส่วนตัวของเรา เช่น:
- เอกสาร PDF ของบริษัท
- ฐานข้อมูล SQL
- ไฟล์ Word/Excel
- ข้อมูลจาก API

LlamaIndex สร้าง **bridge** ระหว่างข้อมูลเหล่านี้กับ LLM โดยใช้เทคนิค **RAG (Retrieval-Augmented Generation)**

### LlamaIndex vs LangChain

| Feature | LlamaIndex | LangChain |
|---|---|---|
| จุดเด่น | Data indexing & retrieval | General-purpose LLM orchestration |
| Use case หลัก | Q&A เหนือ documents | Chains, Agents, Tools |
| Learning curve | ง่ายกว่า | ซับซ้อนกว่า |
| ความลึกใน RAG | ลึกมาก | ปานกลาง |

---

## 2. ติดตั้ง LlamaIndex

```bash
# ติดตั้ง core package
pip install llama-index

# ติดตั้ง integrations เพิ่มเติม (แนะนำ)
pip install llama-index-llms-openai          # OpenAI LLM
pip install llama-index-embeddings-openai    # OpenAI Embeddings
pip install llama-index-vector-stores-chroma # ChromaDB
pip install llama-index-readers-file         # File readers

# หรือติดตั้งทีเดียว (สำหรับเริ่มต้น)
pip install llama-index llama-index-llms-openai llama-index-embeddings-openai
```

### ตั้งค่า API Key

```python
import os
os.environ["OPENAI_API_KEY"] = "sk-..."

# หรือใช้ python-dotenv
from dotenv import load_dotenv
load_dotenv()  # อ่านจาก .env file
```

---

## 3. Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                   YOUR APPLICATION                   │
└────────────────────────┬────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────┐
│                    LLAMAINDEX                        │
│                                                      │
│  ┌──────────┐    ┌──────────┐    ┌──────────────┐   │
│  │  Data    │    │  Index   │    │    Query     │   │
│  │ Loaders  │───▶│ (Vector/ │───▶│   Engine /  │   │
│  │(Readers) │    │  Graph)  │    │  Chat Engine │   │
│  └──────────┘    └──────────┘    └──────┬───────┘   │
│                                         │            │
│  ┌──────────────────────────────────────▼─────────┐ │
│  │            LLM + Embeddings                    │ │
│  │    (OpenAI / Anthropic / Ollama / ...)         │ │
│  └────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

### 5 Building Blocks หลัก

1. **Documents & Nodes** — หน่วยข้อมูลพื้นฐาน
2. **Loaders** — อ่านข้อมูลจากแหล่งต่างๆ
3. **Indexes** — จัดเก็บและค้นหาข้อมูล
4. **Query Engines** — ตอบคำถามจาก Index
5. **LLMs & Embeddings** — โมเดลภาษาและ vector representation

---

## 4. Core Concepts: Documents & Nodes

### Document

`Document` คือ **หน่วยข้อมูลหลัก** ที่ LlamaIndex ใช้งาน ประกอบด้วย:
- `text` — เนื้อหาข้อมูล
- `metadata` — ข้อมูลเพิ่มเติม (filename, author, date, ...)
- `doc_id` — unique identifier

```python
from llama_index.core import Document

# สร้าง Document ด้วยตัวเอง
doc = Document(
    text="LlamaIndex คือ framework สำหรับสร้าง RAG applications",
    metadata={
        "source": "tutorial",
        "author": "john",
        "date": "2024-01-01"
    }
)

print(doc.text)       # LlamaIndex คือ framework...
print(doc.metadata)   # {'source': 'tutorial', ...}
print(doc.doc_id)     # auto-generated UUID
```

### Node

`Node` คือ **ชิ้นส่วนย่อย (chunk)** ของ Document หลังจากที่ถูกแบ่ง (split)

```python
from llama_index.core.schema import TextNode

# สร้าง Node โดยตรง
node = TextNode(
    text="ส่วนหนึ่งของเอกสาร...",
    metadata={"page": 1, "source": "document.pdf"},
    id_="node-001"
)

# Node มี relationships กับ Document
node.relationships  # เก็บ parent document reference
```

### Document → Nodes Flow

```
Document (ทั้งไฟล์)
    │
    ├── parse/split ด้วย NodeParser
    │
    ▼
Node 1 (chunk 1)  ←→  Node 2 (chunk 2)  ←→  Node 3 (chunk 3)
  [text + meta]         [text + meta]          [text + meta]
```

---

## 5. Settings: ตั้งค่า Global Configuration

ใน LlamaIndex v0.10+ ใช้ `Settings` object แทน `ServiceContext` (deprecated)

```python
from llama_index.core import Settings
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding

# ตั้งค่า LLM
Settings.llm = OpenAI(
    model="gpt-4o",
    temperature=0.1,
    max_tokens=2048
)

# ตั้งค่า Embedding model
Settings.embed_model = OpenAIEmbedding(
    model="text-embedding-3-small"
)

# ตั้งค่าการแบ่ง chunk
Settings.chunk_size = 1024       # ขนาด chunk (tokens)
Settings.chunk_overlap = 20      # overlap ระหว่าง chunks

# ตั้งค่า context window
Settings.context_window = 4096
Settings.num_output = 256
```

### ใช้ LLM อื่นๆ

```python
# Anthropic Claude
from llama_index.llms.anthropic import Anthropic
Settings.llm = Anthropic(model="claude-3-5-sonnet-20241022")

# Ollama (Local)
from llama_index.llms.ollama import Ollama
Settings.llm = Ollama(model="llama3.1", request_timeout=120.0)

# Azure OpenAI
from llama_index.llms.azure_openai import AzureOpenAI
Settings.llm = AzureOpenAI(
    engine="gpt-4",
    model="gpt-4",
    azure_endpoint="https://YOUR_RESOURCE.openai.azure.com/",
    api_key="YOUR_KEY",
    api_version="2024-02-01"
)
```

### ใช้ Embedding อื่นๆ

```python
# HuggingFace (ฟรี, ทำงาน local)
from llama_index.embeddings.huggingface import HuggingFaceEmbedding
Settings.embed_model = HuggingFaceEmbedding(
    model_name="BAAI/bge-small-en-v1.5"
)

# Ollama Embeddings
from llama_index.embeddings.ollama import OllamaEmbedding
Settings.embed_model = OllamaEmbedding(model_name="nomic-embed-text")
```

---

## 6. Hello World: RAG ง่ายที่สุด

```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader, Settings
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding
import os

# 0. ตั้งค่า
os.environ["OPENAI_API_KEY"] = "sk-..."

Settings.llm = OpenAI(model="gpt-4o-mini")
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")

# 1. โหลดเอกสาร (จากโฟลเดอร์ ./data/)
documents = SimpleDirectoryReader("./data").load_data()
print(f"Loaded {len(documents)} documents")

# 2. สร้าง Index (แปลง docs → embeddings → เก็บใน vector store)
index = VectorStoreIndex.from_documents(documents)

# 3. สร้าง Query Engine
query_engine = index.as_query_engine()

# 4. ถามคำถาม
response = query_engine.query("สรุปเนื้อหาหลักของเอกสาร")
print(response)
```

นั่นแหล่ะ! 4 บรรทัดหลักทำ RAG ได้เลย 🎉

---

## 7. Understanding the Pipeline

ทุกครั้งที่เรียก `.query()`:

```
User Query: "สรุปเนื้อหาหลัก"
     │
     ▼
1. [Embed Query]
   "สรุปเนื้อหาหลัก" → vector [0.12, -0.34, 0.89, ...]
     │
     ▼
2. [Retrieve]
   ค้นหา Top-K nodes ที่ใกล้เคียงที่สุดใน vector store
   (default: k=2)
     │
     ▼
3. [Synthesize]
   Context = Node1.text + Node2.text
   Prompt = "Answer based on context: {context}\n\nQuestion: {query}"
     │
     ▼
4. [LLM Response]
   GPT-4o → "เนื้อหาหลักของเอกสารคือ..."
     │
     ▼
Response object (with source nodes)
```

---

## 8. Response Object

```python
response = query_engine.query("LlamaIndex คืออะไร?")

# ข้อความตอบกลับ
print(str(response))
print(response.response)

# Source nodes (chunks ที่ใช้ตอบ)
for node in response.source_nodes:
    print(f"Score: {node.score:.3f}")
    print(f"Text: {node.text[:200]}")
    print(f"Metadata: {node.metadata}")
    print("---")

# Raw LLM response
print(response.raw)
```

---

## 9. Prompt Templates

LlamaIndex ใช้ prompt templates ในการสื่อสารกับ LLM

```python
from llama_index.core import PromptTemplate

# Default QA prompt (ดูว่า LlamaIndex ใช้อะไร)
query_engine = index.as_query_engine()
prompts = query_engine.get_prompts()
for key, prompt in prompts.items():
    print(f"\n=== {key} ===")
    print(prompt.get_template())
```

```python
# Custom prompt template
qa_prompt = PromptTemplate(
    """คุณเป็นผู้ช่วย AI ที่ตอบคำถามเป็นภาษาไทย
    
ข้อมูลที่เกี่ยวข้อง:
---------------------
{context_str}
---------------------

คำถาม: {query_str}

กรุณาตอบคำถามโดยอ้างอิงจากข้อมูลข้างต้นเท่านั้น 
ถ้าไม่มีข้อมูลเพียงพอ ให้บอกว่า "ไม่มีข้อมูลเพียงพอ"

คำตอบ:"""
)

query_engine = index.as_query_engine(
    text_qa_template=qa_prompt
)

response = query_engine.query("LlamaIndex ใช้ทำอะไร?")
```

---

## 10. Streaming Response

```python
# Streaming (แสดงผลทีละ token)
query_engine = index.as_query_engine(streaming=True)

streaming_response = query_engine.query("อธิบาย LlamaIndex")

# แสดงทีละ chunk
for text in streaming_response.response_gen:
    print(text, end="", flush=True)
print()  # newline
```

---

## สรุป Phase 1

| Concept | คืออะไร |
|---|---|
| Document | หน่วยข้อมูลหลัก (ทั้งไฟล์) |
| Node | ชิ้นส่วนย่อยของ Document (chunk) |
| Settings | global config สำหรับ LLM + Embeddings |
| VectorStoreIndex | Index ที่ใช้ vector similarity search |
| QueryEngine | interface สำหรับถามคำถาม |
| Response | object ที่ contains คำตอบ + source nodes |

---

## ขั้นตอนถัดไป

👉 **Phase 2: Data Ingestion & Indexing** — เรียนรู้วิธีโหลดข้อมูลจากแหล่งต่างๆ (PDF, SQL, Web, API), Node Parsers, และประเภทของ Index

---

*Phase 1/4 | LlamaIndex Deep Dive Series*
