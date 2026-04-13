# RAG Deep Dive — Retrieval-Augmented Generation ทุกเทคนิค

> RAG คือเทคนิคที่เสริม LLM ด้วยข้อมูลจากแหล่งภายนอก
> ทำให้ AI ตอบจากข้อมูลจริง ไม่ใช่แค่ "จำ" จากตอนฝึก

---

## สารบัญ

1. [RAG คืออะไร — ทำไมต้องใช้](#1-rag-คืออะไร)
2. [Architecture & Pipeline](#2-architecture)
3. [Step 1: Document Loading — โหลดข้อมูล](#3-document-loading)
4. [Step 2: Chunking — ตัดเอกสาร](#4-chunking)
5. [Step 3: Embedding — แปลงเป็น Vector](#5-embedding)
6. [Step 4: Vector Store — จัดเก็บ](#6-vector-store)
7. [Step 5: Retrieval — ค้นหา](#7-retrieval)
8. [Step 6: Generation — สร้างคำตอบ](#8-generation)
9. [RAG แบบสมบูรณ์ — End-to-End Example](#9-end-to-end)
10. [Advanced RAG Techniques](#10-advanced-techniques)
11. [RAG Agent with LangGraph](#11-rag-agent)
12. [Evaluation — วัดคุณภาพ RAG](#12-evaluation)
13. [Production Best Practices](#13-best-practices)

---

## 1. RAG คืออะไร

### ปัญหาของ LLM ที่ RAG แก้ได้

```
❌ ปัญหา                              ✅ RAG แก้ได้
───────────────────────────────────────────────────────────
Knowledge cutoff (ข้อมูลเก่า)         → ดึงข้อมูลล่าสุดมาให้
ไม่รู้ข้อมูลภายใน (บริษัท, FAQ)       → ค้นหาจากเอกสารภายใน
Hallucination (ตอบข้อมูลไม่จริง)     → ตอบจาก context ที่ดึงมา
ไม่สามารถอ้างอิงแหล่งที่มา           → บอก source ได้
Context window จำกัด                  → ดึงแค่ส่วนที่เกี่ยวข้อง
```

### RAG vs Fine-tuning

| | RAG | Fine-tuning |
|---|---|---|
| **ข้อมูลเปลี่ยนบ่อย** | ✅ อัปเดตได้ทันที | ❌ ต้อง train ใหม่ |
| **ต้องอ้างอิง source** | ✅ ทำได้ | ❌ ทำไม่ได้ |
| **ข้อมูลลับ** | ✅ ข้อมูลอยู่ในระบบเรา | ⚠️ ต้องส่งไป train |
| **ความแม่นยำสูง** | ✅ ตอบจากข้อมูลจริง | ⚠️ อาจ hallucinate |
| **ค่าใช้จ่าย** | 💰 ค่า embedding + storage | 💰💰💰 ค่า training |
| **เหมาะกับ** | Q&A, search, knowledge base | ปรับ style, เรียนรู้ pattern |

---

## 2. Architecture

### RAG Pipeline แบบละเอียด

```
═══════════════════════════════════════════════════════════
                    INDEXING (ทำครั้งเดียว)
═══════════════════════════════════════════════════════════

[PDF] [TXT] [CSV] [Web] [DB]        ← แหล่งข้อมูล
          │
          ▼
    ┌─────────────┐
    │   LOADING    │                 ← โหลดเอกสาร
    │  (Loaders)   │
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  SPLITTING   │                 ← ตัดเป็น chunks
    │ (Splitters)  │
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  EMBEDDING   │                 ← แปลงเป็น vectors
    │   (Model)    │
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │   STORING    │                 ← เก็บใน Vector Store
    │ (VectorDB)   │
    └─────────────┘

═══════════════════════════════════════════════════════════
                   QUERYING (ทุกครั้งที่ถาม)
═══════════════════════════════════════════════════════════

    User Question
          │
          ▼
    ┌─────────────┐
    │  EMBEDDING   │                 ← แปลง query เป็น vector
    │   (Model)    │
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  RETRIEVAL   │                 ← ค้นหา chunks ที่คล้าย
    │ (VectorDB)   │
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  GENERATION  │                 ← LLM สร้างคำตอบ
    │    (LLM)     │                    จาก context + question
    └──────┬──────┘
           │
           ▼
      Final Answer
    (+ source references)
```

---

## 3. Document Loading

### 3.1 รวม Loaders ที่ใช้บ่อย

```bash
pip install pypdf docx2txt beautifulsoup4 unstructured
```

```python
# === PDF ===
from langchain_community.document_loaders import PyPDFLoader

loader = PyPDFLoader("docs/manual.pdf")
docs = loader.load()
# แต่ละหน้า = 1 Document
# metadata: {'source': 'docs/manual.pdf', 'page': 0}


# === Text ===
from langchain_community.document_loaders import TextLoader

loader = TextLoader("docs/readme.txt", encoding="utf-8")
docs = loader.load()


# === CSV ===
from langchain_community.document_loaders.csv_loader import CSVLoader

loader = CSVLoader("data/products.csv", encoding="utf-8")
docs = loader.load()
# แต่ละ row = 1 Document


# === Word (.docx) ===
from langchain_community.document_loaders import Docx2txtLoader

loader = Docx2txtLoader("docs/report.docx")
docs = loader.load()


# === Website ===
from langchain_community.document_loaders import WebBaseLoader

loader = WebBaseLoader(["https://example.com/page1", "https://example.com/page2"])
docs = loader.load()


# === JSON ===
from langchain_community.document_loaders import JSONLoader

loader = JSONLoader(
    file_path="data/faq.json",
    jq_schema=".[]",
    content_key="answer",
    metadata_func=lambda record, meta: {**meta, "question": record["question"]}
)
docs = loader.load()


# === Directory (โหลดทุกไฟล์ใน folder) ===
from langchain_community.document_loaders import DirectoryLoader

loader = DirectoryLoader("docs/", glob="**/*.pdf", show_progress=True)
docs = loader.load()
```

### 3.2 เพิ่ม Metadata ที่มีประโยชน์

```python
from langchain_core.documents import Document

# เพิ่ม metadata เอง เพื่อใช้ filter ตอน retrieval
docs = [
    Document(
        page_content="นโยบายลาพักร้อน: พนักงานมีสิทธิ์ลา 15 วัน/ปี",
        metadata={
            "source": "hr-policy-2025.pdf",
            "department": "HR",
            "category": "leave-policy",
            "version": "2025",
            "last_updated": "2025-01-01",
        }
    ),
]
```

---

## 4. Chunking

### 4.1 ทำไมต้อง Chunk ?

- Embedding model ทำงานดีกับ text สั้น (300-500 คำ)
- ค้นหาได้แม่นยำกว่า (เจอ chunk ที่เกี่ยวข้องตรงประเด็น)
- ประหยัด token ที่ส่งให้ LLM (ส่งแค่ chunks ที่เกี่ยวข้อง)

### 4.2 RecursiveCharacterTextSplitter (แนะนำสำหรับ text ทั่วไป)

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,       # ขนาดสูงสุด (ตัวอักษร)
    chunk_overlap=100,    # ซ้อนทับเพื่อรักษาบริบท
    separators=["\n\n", "\n", ".", " ", ""],
    length_function=len,
)

# Split text ธรรมดา
chunks = splitter.split_text(long_text)

# Split documents (รักษา metadata)
doc_chunks = splitter.split_documents(documents)
```

### 4.3 Markdown Splitter (สำหรับ Markdown)

```python
from langchain_text_splitters import MarkdownHeaderTextSplitter

# ตัดตาม header ของ Markdown
headers_to_split = [
    ("#", "Header 1"),
    ("##", "Header 2"),
    ("###", "Header 3"),
]

md_splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers_to_split)

md_text = """
# บทที่ 1: แนะนำ LangChain
LangChain คือ framework สำหรับ AI...

## 1.1 ติดตั้ง
pip install langchain...

## 1.2 เริ่มต้นใช้งาน
สร้างโมเดลแรก...

# บทที่ 2: Models
อธิบายเรื่อง Models...
"""

chunks = md_splitter.split_text(md_text)
for chunk in chunks:
    print(f"Content: {chunk.page_content[:50]}...")
    print(f"Metadata: {chunk.metadata}")
    print()
```

**Output:**
```
Content: LangChain คือ framework สำหรับ AI...
Metadata: {'Header 1': 'บทที่ 1: แนะนำ LangChain'}

Content: pip install langchain...
Metadata: {'Header 1': 'บทที่ 1: แนะนำ LangChain', 'Header 2': '1.1 ติดตั้ง'}

Content: สร้างโมเดลแรก...
Metadata: {'Header 1': 'บทที่ 1: แนะนำ LangChain', 'Header 2': '1.2 เริ่มต้นใช้งาน'}

Content: อธิบายเรื่อง Models...
Metadata: {'Header 1': 'บทที่ 2: Models'}
```

### 4.4 Code Splitter (สำหรับ source code)

```python
from langchain_text_splitters import Language, RecursiveCharacterTextSplitter

python_splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.PYTHON,
    chunk_size=500,
    chunk_overlap=50,
)

# รองรับหลายภาษา:
# Language.PYTHON, Language.JAVASCRIPT, Language.TYPESCRIPT,
# Language.JAVA, Language.GO, Language.RUST, Language.HTML, etc.
```

### 4.5 Semantic Chunking (ตัดตามความหมาย)

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_google_genai import GoogleGenerativeAIEmbeddings

embeddings = GoogleGenerativeAIEmbeddings(model="models/text-embedding-004")

semantic_splitter = SemanticChunker(
    embeddings,
    breakpoint_threshold_type="percentile",
    breakpoint_threshold_amount=95,
)

# จะตัดเมื่อ "ความหมาย" เปลี่ยนไปมาก
chunks = semantic_splitter.split_text(long_text)
```

### 4.6 เลือก Strategy

| เอกสาร | Splitter แนะนำ | chunk_size | overlap |
|--------|----------------|------------|---------|
| เอกสารทั่วไป | RecursiveCharacter | 500 | 100 |
| FAQ (Q&A pairs) | อย่า split (1 Q&A = 1 chunk) | — | — |
| Markdown / Wiki | MarkdownHeader + Recursive | 500 | 100 |
| Source Code | Language-specific | 500 | 50 |
| กฎหมาย / สัญญา | RecursiveCharacter | 300 | 100 |
| บทความยาว | SemanticChunker | auto | auto |

---

## 5. Embedding

### 5.1 Embedding Models

```python
# === Google ===
from langchain_google_genai import GoogleGenerativeAIEmbeddings

embeddings = GoogleGenerativeAIEmbeddings(
    model="models/text-embedding-004",
    # task_type="retrieval_document"  # สำหรับ indexing
    # task_type="retrieval_query"     # สำหรับ query
)

# === OpenAI ===
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(
    model="text-embedding-3-small",  # ถูกกว่า
    # model="text-embedding-3-large",  # แม่นกว่า
)

# === Hugging Face (ฟรี, รันบนเครื่อง) ===
from langchain_huggingface import HuggingFaceEmbeddings

embeddings = HuggingFaceEmbeddings(
    model_name="sentence-transformers/all-MiniLM-L6-v2",
    model_kwargs={"device": "cpu"},  # หรือ "cuda" ถ้ามี GPU
)
```

### 5.2 ใช้งาน Embedding

```python
# Embed ข้อความเดียว (สำหรับ query)
query_vector = embeddings.embed_query("LangChain คืออะไร?")
print(f"Dimensions: {len(query_vector)}")  # 768 (ขึ้นกับ model)

# Embed หลายข้อความ (สำหรับ documents)
doc_vectors = embeddings.embed_documents([
    "LangChain คือ Framework",
    "RAG คือเทคนิคดึงข้อมูล",
])

# คำนวณ similarity ด้วยตัวเอง
import numpy as np

def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

sim = cosine_similarity(query_vector, doc_vectors[0])
print(f"Similarity: {sim:.4f}")  # ยิ่งใกล้ 1 ยิ่งคล้าย
```

### 5.3 เปรียบเทียบ Embedding Models

| Model | Dimensions | ราคา | ความแม่นยำ | เหมาะกับ |
|-------|-----------|------|-----------|---------|
| text-embedding-004 (Google) | 768 | ฟรี 1500 req/min | ดี | ทั่วไป |
| text-embedding-3-small (OpenAI) | 1536 | $0.02/1M tokens | ดี | ราคาประหยัด |
| text-embedding-3-large (OpenAI) | 3072 | $0.13/1M tokens | ดีมาก | ต้องการแม่นยำ |
| all-MiniLM-L6-v2 (HF) | 384 | ฟรี (local) | ปานกลาง | Offline/ทดสอบ |

---

## 6. Vector Store

### 6.1 Chroma (แนะนำสำหรับเริ่มต้น)

```bash
pip install langchain-chroma
```

```python
from langchain_chroma import Chroma

# === สร้าง Vector Store จากเอกสาร ===
vectorstore = Chroma.from_documents(
    documents=doc_chunks,           # chunks ที่ split แล้ว
    embedding=embeddings,           # embedding model
    collection_name="my-docs",
    persist_directory="./chroma_db", # บันทึกลงดิสก์
)

# === โหลด Vector Store ที่มีอยู่แล้ว ===
vectorstore = Chroma(
    collection_name="my-docs",
    embedding_function=embeddings,
    persist_directory="./chroma_db",
)

# === เพิ่มเอกสารเข้าไปทีหลัง ===
vectorstore.add_documents(new_documents)

# === ลบเอกสาร ===
vectorstore.delete(ids=["doc-id-1", "doc-id-2"])

# === ดูจำนวนเอกสาร ===
print(vectorstore._collection.count())
```

### 6.2 FAISS (เร็ว, ใช้ RAM)

```bash
pip install faiss-cpu  # หรือ faiss-gpu
```

```python
from langchain_community.vectorstores import FAISS

# สร้าง
vectorstore = FAISS.from_documents(doc_chunks, embeddings)

# บันทึก / โหลด
vectorstore.save_local("faiss_index")
vectorstore = FAISS.load_local("faiss_index", embeddings, allow_dangerous_deserialization=True)
```

### 6.3 เปรียบเทียบ Vector Stores

| Vector Store | ประเภท | เหมาะกับ | จุดเด่น |
|---|---|---|---|
| **Chroma** | Embedded / Server | Development, Small-Medium | ง่าย, persist ได้ |
| **FAISS** | In-memory | Large dataset, Speed | เร็วมาก |
| **Pinecone** | Cloud managed | Production | Scalable, managed |
| **Weaviate** | Self-hosted/Cloud | Production | Hybrid search |
| **Qdrant** | Self-hosted/Cloud | Production | Performance |
| **pgvector** | PostgreSQL extension | มี Postgres อยู่แล้ว | ไม่ต้องเพิ่ม infra |

---

## 7. Retrieval

### 7.1 Basic Retrieval

```python
# สร้าง retriever
retriever = vectorstore.as_retriever(
    search_type="similarity",   # ค้นหาด้วยความคล้าย
    search_kwargs={"k": 4},     # ดึง 4 chunks
)

# ค้นหา
docs = retriever.invoke("LangChain คืออะไร?")
for doc in docs:
    print(f"[{doc.metadata.get('source', '?')}] {doc.page_content[:100]}...")
```

### 7.2 Search Types

```python
# === 1. Similarity Search (Default) ===
# ค้นหา chunks ที่ vector ใกล้ query มากที่สุด
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 4}
)

# === 2. MMR (Maximal Marginal Relevance) ===
# ค้นหาแบบมีความหลากหลาย ไม่เอาแต่ chunks ที่คล้ายกัน
retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={
        "k": 4,           # จำนวนผลลัพธ์สุดท้าย
        "fetch_k": 20,    # ดึงมาก่อน 20 แล้วเลือก 4 ที่หลากหลาย
        "lambda_mult": 0.5, # 0 = หลากหลายสุด, 1 = คล้ายสุด
    }
)

# === 3. Similarity Score Threshold ===
# ค้นหาเฉพาะ chunks ที่มี score สูงกว่า threshold
retriever = vectorstore.as_retriever(
    search_type="similarity_score_threshold",
    search_kwargs={
        "score_threshold": 0.5,  # เอาเฉพาะ score > 0.5
        "k": 10,
    }
)
```

### 7.3 Metadata Filtering

```python
# ค้นหาเฉพาะเอกสารจาก department HR
docs = vectorstore.similarity_search(
    "นโยบายลา",
    k=4,
    filter={"department": "HR"}
)

# หลายเงื่อนไข (ขึ้นกับ vector store)
# Chroma รองรับ:
docs = vectorstore.similarity_search(
    "นโยบาย",
    filter={
        "$and": [
            {"department": "HR"},
            {"version": "2025"},
        ]
    }
)
```

### 7.4 Multi-Query Retriever

ให้ LLM สร้างคำถามหลายมุมมองแล้วค้นหาทุกมุม:

```python
from langchain.retrievers.multi_query import MultiQueryRetriever

multi_retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(search_kwargs={"k": 3}),
    llm=model,
)

# คำถามเดียว → LLM สร้าง 3 คำถามที่แตกต่าง → ค้นหา 3 ครั้ง → รวม
docs = multi_retriever.invoke("LangGraph ใช้ทำอะไร?")

# ภายใน LLM จะสร้าง:
# 1. "LangGraph คืออะไร?"
# 2. "LangGraph มีประโยชน์อย่างไร?"
# 3. "ใช้ LangGraph สร้างอะไรได้บ้าง?"
# แล้วค้นหาทั้ง 3 คำถาม → รวมผลลัพธ์ → deduplicate
```

### 7.5 Contextual Compression

ตัดส่วนที่ไม่เกี่ยวข้องออกจาก chunk:

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import LLMChainExtractor

compressor = LLMChainExtractor.from_llm(model)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 5}),
)

# chunk เดิม: "บริษัท ABC ก่อตั้งปี 2010 มีพนักงาน 500 คน สำนักงานใหญ่อยู่กรุงเทพ"
# ถ้าถาม "บริษัท ABC มีพนักงานกี่คน?"
# compressed: "มีพนักงาน 500 คน"
```

### 7.6 Ensemble Retriever (Hybrid Search)

รวม keyword search + semantic search:

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

# Keyword-based retriever (BM25)
bm25_retriever = BM25Retriever.from_documents(doc_chunks)
bm25_retriever.k = 4

# Semantic retriever
semantic_retriever = vectorstore.as_retriever(search_kwargs={"k": 4})

# รวมกัน (weighted)
ensemble_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, semantic_retriever],
    weights=[0.4, 0.6],  # ให้น้ำหนัก semantic มากกว่า
)

docs = ensemble_retriever.invoke("วิธีสมัครใช้งาน")
```

---

## 8. Generation

### 8.1 RAG Prompt พื้นฐาน

```python
from langchain_core.prompts import ChatPromptTemplate

rag_prompt = ChatPromptTemplate.from_messages([
    ("system", """คุณคือผู้ช่วย AI ที่ตอบคำถามจากข้อมูลที่ให้มาเท่านั้น

กฎสำคัญ:
1. ตอบจากข้อมูลอ้างอิงที่ให้มาเท่านั้น ห้ามแต่งเพิ่ม
2. ถ้าไม่มีข้อมูลเพียงพอ ให้บอกตรงๆ ว่า "ไม่มีข้อมูลในเอกสาร"
3. อ้างอิงแหล่งที่มาเสมอ
4. ตอบเป็นภาษาไทย กระชับ ได้ใจความ

ข้อมูลอ้างอิง:
{context}"""),

    ("human", "{question}"),
])
```

### 8.2 RAG Chain ด้วย LCEL

```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

def format_docs(docs):
    """แปลง docs เป็น text สำหรับ context"""
    formatted = []
    for i, doc in enumerate(docs, 1):
        source = doc.metadata.get("source", "unknown")
        formatted.append(f"[{i}] ({source})\n{doc.page_content}")
    return "\n\n---\n\n".join(formatted)

# สร้าง RAG Chain
rag_chain = (
    {
        "context": retriever | format_docs,   # ดึง docs → format
        "question": RunnablePassthrough(),     # ส่ง question ต่อ
    }
    | rag_prompt        # สร้าง prompt
    | model             # ส่งให้ LLM
    | StrOutputParser() # ดึงแค่ text
)

# ใช้งาน
answer = rag_chain.invoke("LangGraph คืออะไร?")
print(answer)
```

### 8.3 RAG Chain ที่ Return Sources ด้วย

```python
from langchain_core.runnables import RunnableParallel

# Chain ที่ return ทั้งคำตอบและ sources
rag_chain_with_sources = RunnableParallel(
    answer=(
        {"context": retriever | format_docs, "question": RunnablePassthrough()}
        | rag_prompt
        | model
        | StrOutputParser()
    ),
    sources=retriever | (lambda docs: [
        {"content": d.page_content[:100], "source": d.metadata.get("source")}
        for d in docs
    ]),
)

result = rag_chain_with_sources.invoke("ลาพักร้อนได้กี่วัน?")
print(f"Answer: {result['answer']}")
print(f"Sources: {result['sources']}")
```

**Output:**
```
Answer: พนักงานมีสิทธิ์ลาพักร้อน 15 วันต่อปีครับ (อ้างอิง: hr-policy-2025.pdf)
Sources: [
  {'content': 'นโยบายลาพักร้อน: พนักงานมีสิทธิ์ลา 15 วัน/ปี...', 'source': 'hr-policy-2025.pdf'}
]
```

---

## 9. End-to-End Example

### ตัวอย่างสมบูรณ์: Knowledge Base Bot สำหรับบริษัท

```python
import os
from dotenv import load_dotenv
from langchain_google_genai import ChatGoogleGenerativeAI, GoogleGenerativeAIEmbeddings
from langchain_chroma import Chroma
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_core.documents import Document
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

load_dotenv()

# ============================================================
# STEP 1: เตรียมข้อมูล (ทำครั้งเดียว)
# ============================================================

documents = [
    # HR Policies
    Document(
        page_content="""นโยบายการลาของบริษัท XYZ (ปี 2025)

ลาพักร้อน: พนักงานทุกคนมีสิทธิ์ลาพักร้อน 15 วันทำการต่อปี
สามารถสะสมวันลาได้ไม่เกิน 5 วัน ต่อไปยังปีถัดไป
ต้องแจ้งล่วงหน้าอย่างน้อย 3 วันทำการ

ลาป่วย: พนักงานมีสิทธิ์ลาป่วยได้ 30 วันทำการต่อปี
ถ้าลาป่วยเกิน 3 วันติดต่อกัน ต้องมีใบรับรองแพทย์

ลากิจ: มีสิทธิ์ลากิจ 5 วันต่อปี โดยได้รับค่าจ้าง
ต้องแจ้งล่วงหน้าอย่างน้อย 1 วัน""",
        metadata={"source": "hr-policy.pdf", "category": "leave", "department": "HR"}
    ),

    Document(
        page_content="""สวัสดิการพนักงาน บริษัท XYZ

ประกันสุขภาพกลุ่ม: ครอบคลุมพนักงานและครอบครัว (คู่สมรส + บุตร)
วงเงินค่ารักษาผู้ป่วยนอก: 3,000 บาท/ครั้ง
วงเงินค่ารักษาผู้ป่วยใน: 100,000 บาท/ครั้ง

ค่าอาหาร: 100 บาท/วันทำการ จ่ายพร้อมเงินเดือน
ค่าเดินทาง: 2,500 บาท/เดือน (เหมาจ่าย)
ค่าโทรศัพท์: 500 บาท/เดือน (สำหรับตำแหน่ง Senior ขึ้นไป)

กองทุนสำรองเลี้ยงชีพ: บริษัทสมทบ 5% ของเงินเดือน
โบนัส: ประเมินประจำปี 0-4 เดือน ตามผลงาน""",
        metadata={"source": "benefits.pdf", "category": "benefits", "department": "HR"}
    ),

    Document(
        page_content="""คู่มือการใช้งานระบบ IT

การเข้าใช้งานระบบ:
- ใช้ email บริษัท (@xyz.com) เป็น username
- รหัสผ่านเริ่มต้น: XYZ + เลขพนักงาน 4 หลัก (เช่น XYZ1234)
- ต้องเปลี่ยนรหัสผ่านภายใน 24 ชั่วโมง

ลืมรหัสผ่าน:
1. ไปที่ https://portal.xyz.com/reset
2. กรอก email บริษัท
3. ระบบจะส่ง link reset ไปทาง email
4. ถ้าไม่ได้รับ email ให้ติดต่อ IT Help Desk: ext. 1234

VPN:
- ดาวน์โหลด FortiClient จาก https://portal.xyz.com/vpn
- Server: vpn.xyz.com
- ใช้ username/password เดียวกับ email

แจ้งปัญหา IT:
- ส่ง ticket ที่ https://helpdesk.xyz.com
- โทร ext. 1234 (จันทร์-ศุกร์ 8:00-18:00)
- ฉุกเฉินนอกเวลา: 081-xxx-xxxx""",
        metadata={"source": "it-manual.pdf", "category": "IT", "department": "IT"}
    ),
]

# STEP 2: Chunking
splitter = RecursiveCharacterTextSplitter(
    chunk_size=400,
    chunk_overlap=80,
    separators=["\n\n", "\n", ".", " "],
)
chunks = splitter.split_documents(documents)
print(f"📄 ตัดเอกสารได้ {len(chunks)} chunks")

# STEP 3: Embedding + Store
embeddings = GoogleGenerativeAIEmbeddings(model="models/text-embedding-004")
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings,
    collection_name="company-kb",
    persist_directory="./company_chroma_db",
)
print(f"✅ สร้าง Vector Store เรียบร้อย")

# ============================================================
# STEP 4: สร้าง RAG System
# ============================================================

retriever = vectorstore.as_retriever(
    search_type="mmr",          # ใช้ MMR เพื่อความหลากหลาย
    search_kwargs={"k": 3, "fetch_k": 10},
)

model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

rag_prompt = ChatPromptTemplate.from_messages([
    ("system", """คุณคือ "น้องเอ็กซ์" ผู้ช่วย AI ของบริษัท XYZ
ตอบคำถามพนักงานจากข้อมูลภายในบริษัทเท่านั้น

กฎ:
1. ตอบจากข้อมูลที่ให้มาเท่านั้น ห้ามแต่งเติม
2. ถ้าไม่มีข้อมูล บอกว่า "ขอโทษค่ะ ไม่มีข้อมูลนี้ในระบบ แนะนำให้ติดต่อ HR โดยตรง"
3. อ้างอิงเอกสารที่ใช้ตอบ
4. ใช้ภาษาสุภาพ เป็นกันเอง ลงท้ายด้วย "ค่ะ"

ข้อมูลอ้างอิง:
{context}"""),
    ("human", "{question}"),
])

def format_docs(docs):
    parts = []
    for i, doc in enumerate(docs, 1):
        src = doc.metadata.get("source", "?")
        cat = doc.metadata.get("category", "?")
        parts.append(f"[{i}] (เอกสาร: {src}, หมวด: {cat})\n{doc.page_content}")
    return "\n\n---\n\n".join(parts)

# สร้าง function สำหรับถาม
def ask(question: str) -> dict:
    # Retrieve
    docs = retriever.invoke(question)

    # Format context
    context = format_docs(docs)

    # Generate
    formatted = rag_prompt.invoke({"context": context, "question": question})
    response = model.invoke(formatted)

    # Collect sources
    sources = list(set([d.metadata.get("source", "?") for d in docs]))

    return {
        "question": question,
        "answer": response.content,
        "sources": sources,
        "num_chunks": len(docs),
    }


# ============================================================
# STEP 5: ทดสอบ
# ============================================================

questions = [
    "ลาพักร้อนได้กี่วัน?",
    "ถ้าลืมรหัสผ่านต้องทำยังไง?",
    "ค่าอาหารต่อเดือนเท่าไหร่ถ้าทำงาน 22 วัน?",
    "ต้องส่งใบรับรองแพทย์ตอนไหน?",
    "VPN ใช้ server อะไร?",
    "มีโบนัสกี่เดือน?",
    "วิธีทำส้มตำ?",  # นอกขอบเขต
]

for q in questions:
    result = ask(q)
    print(f"❓ {result['question']}")
    print(f"💬 {result['answer']}")
    print(f"📎 Sources: {result['sources']}")
    print(f"📊 Chunks used: {result['num_chunks']}")
    print("=" * 60)
    print()
```

**Output ตัวอย่าง:**
```
📄 ตัดเอกสารได้ 9 chunks
✅ สร้าง Vector Store เรียบร้อย

❓ ลาพักร้อนได้กี่วัน?
💬 พนักงานมีสิทธิ์ลาพักร้อน 15 วันทำการต่อปีค่ะ
   และสามารถสะสมวันลาได้ไม่เกิน 5 วันต่อไปยังปีถัดไป
   โดยต้องแจ้งล่วงหน้าอย่างน้อย 3 วันทำการนะคะ
📎 Sources: ['hr-policy.pdf']
📊 Chunks used: 3
============================================================

❓ ถ้าลืมรหัสผ่านต้องทำยังไง?
💬 ถ้าลืมรหัสผ่าน ทำตามขั้นตอนนี้ได้เลยค่ะ:
   1. ไปที่ https://portal.xyz.com/reset
   2. กรอก email บริษัท
   3. ระบบจะส่ง link reset ไปทาง email
   4. ถ้าไม่ได้รับ email ให้ติดต่อ IT Help Desk: ext. 1234
   (อ้างอิง: it-manual.pdf)
📎 Sources: ['it-manual.pdf']
📊 Chunks used: 3
============================================================

❓ วิธีทำส้มตำ?
💬 ขอโทษค่ะ ไม่มีข้อมูลเรื่องวิธีทำส้มตำในระบบ
   น้องเอ็กซ์ตอบได้เฉพาะเรื่องภายในบริษัท XYZ เท่านั้นค่ะ
📎 Sources: ['hr-policy.pdf', 'benefits.pdf']
📊 Chunks used: 3
============================================================
```

---

## 10. Advanced Techniques

### 10.1 Parent Document Retriever

ค้นหาจาก chunk เล็ก แต่ส่ง document ใหญ่ให้ LLM:

```python
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore

# child splitter (เล็ก, สำหรับค้นหา)
child_splitter = RecursiveCharacterTextSplitter(chunk_size=200, chunk_overlap=50)

# parent splitter (ใหญ่, สำหรับส่ง LLM)
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)

store = InMemoryStore()

parent_retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,
    docstore=store,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)

# Index
parent_retriever.add_documents(documents)

# ค้นหา → ได้ parent document (ใหญ่กว่า, มีบริบทมากกว่า)
docs = parent_retriever.invoke("ลาพักร้อน")
```

### 10.2 Self-Query Retriever

ให้ LLM แปลงคำถามเป็น structured query + metadata filter:

```python
from langchain.retrievers.self_query.base import SelfQueryRetriever
from langchain.chains.query_constructor.schema import AttributeInfo

metadata_field_info = [
    AttributeInfo(name="category", description="หมวดหมู่: leave, benefits, IT", type="string"),
    AttributeInfo(name="department", description="แผนก: HR, IT", type="string"),
    AttributeInfo(name="source", description="ชื่อไฟล์เอกสาร", type="string"),
]

self_query_retriever = SelfQueryRetriever.from_llm(
    llm=model,
    vectorstore=vectorstore,
    document_contents="เอกสารภายในบริษัท XYZ เกี่ยวกับนโยบาย HR และ IT",
    metadata_field_info=metadata_field_info,
)

# คำถาม: "นโยบายลาของ HR"
# LLM จะแปลงเป็น: query="นโยบายลา", filter={"department": "HR"}
docs = self_query_retriever.invoke("นโยบายลาของ HR")
```

### 10.3 Re-ranking

จัดอันดับผลลัพธ์ใหม่ด้วย LLM:

```python
from langchain.retrievers.document_compressors import CrossEncoderReranker
from langchain.retrievers import ContextualCompressionRetriever

# ใช้ Cross-Encoder rerank
from langchain_community.cross_encoders import HuggingFaceCrossEncoder

reranker = HuggingFaceCrossEncoder(model_name="cross-encoder/ms-marco-MiniLM-L-6-v2")
compressor = CrossEncoderReranker(model=reranker, top_n=3)

reranking_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 10}),
)

# ดึงมา 10 → rerank → เลือก top 3
docs = reranking_retriever.invoke("สวัสดิการอะไรบ้าง?")
```

### 10.4 Conversational RAG (RAG ที่จำบทสนทนา)

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

# Prompt ที่มี chat history
conversational_rag_prompt = ChatPromptTemplate.from_messages([
    ("system", """ตอบจากข้อมูลอ้างอิงเท่านั้น

ข้อมูลอ้างอิง:
{context}"""),
    MessagesPlaceholder(variable_name="chat_history"),
    ("human", "{question}"),
])

# Condense question prompt
# แปลงคำถาม follow-up ให้เป็นคำถามที่สมบูรณ์
condense_prompt = ChatPromptTemplate.from_messages([
    ("system", "ดูประวัติบทสนทนาและคำถามล่าสุด แปลงเป็นคำถามที่สมบูรณ์ในตัวเอง ตอบแค่คำถามที่แปลงแล้ว"),
    MessagesPlaceholder(variable_name="chat_history"),
    ("human", "{question}"),
])

class ConversationalRAG:
    def __init__(self, retriever, model):
        self.retriever = retriever
        self.model = model
        self.chat_history = []

    def ask(self, question: str) -> str:
        # ถ้ามี history → condense question ก่อน
        if self.chat_history:
            condensed = condense_prompt.invoke({
                "chat_history": self.chat_history,
                "question": question,
            })
            search_query = self.model.invoke(condensed).content
        else:
            search_query = question

        # Retrieve
        docs = self.retriever.invoke(search_query)
        context = "\n\n".join([d.page_content for d in docs])

        # Generate
        formatted = conversational_rag_prompt.invoke({
            "context": context,
            "chat_history": self.chat_history,
            "question": question,
        })
        response = self.model.invoke(formatted)

        # อัปเดต history
        self.chat_history.append({"role": "user", "content": question})
        self.chat_history.append({"role": "assistant", "content": response.content})

        return response.content

# ใช้งาน
rag = ConversationalRAG(retriever, model)
print(rag.ask("ลาพักร้อนได้กี่วัน?"))
# → "พนักงานมีสิทธิ์ลาพักร้อน 15 วัน..."

print(rag.ask("แล้วสะสมวันลาได้ไหม?"))
# → AI จำได้ว่ากำลังคุยเรื่องลาพักร้อน
# → "สามารถสะสมวันลาพักร้อนได้ไม่เกิน 5 วัน..."

print(rag.ask("ต้องแจ้งล่วงหน้ากี่วัน?"))
# → "ต้องแจ้งล่วงหน้าอย่างน้อย 3 วันทำการ..."
```

---

## 11. RAG Agent with LangGraph

RAG + Agent + Tools ที่ตัดสินใจเองว่าจะค้นหาจากแหล่งไหน:

```python
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langchain.tools import tool
from typing import Annotated
from typing_extensions import TypedDict

class RAGAgentState(TypedDict):
    messages: Annotated[list, add_messages]

@tool
def search_hr_docs(query: str) -> str:
    """ค้นหาข้อมูลจากเอกสาร HR เช่น นโยบายลา สวัสดิการ"""
    hr_retriever = vectorstore.as_retriever(
        search_kwargs={"k": 3, "filter": {"department": "HR"}}
    )
    docs = hr_retriever.invoke(query)
    return "\n".join([d.page_content for d in docs]) or "ไม่พบข้อมูล HR"

@tool
def search_it_docs(query: str) -> str:
    """ค้นหาข้อมูลจากเอกสาร IT เช่น วิธีใช้ระบบ VPN รหัสผ่าน"""
    it_retriever = vectorstore.as_retriever(
        search_kwargs={"k": 3, "filter": {"department": "IT"}}
    )
    docs = it_retriever.invoke(query)
    return "\n".join([d.page_content for d in docs]) or "ไม่พบข้อมูล IT"

@tool
def calculate(expression: str) -> str:
    """คำนวณตัวเลข เช่น 100 * 22 หรือ 2500 + 500"""
    try:
        return f"{expression} = {eval(expression)}"
    except:
        return "คำนวณไม่ได้"

tools = [search_hr_docs, search_it_docs, calculate]
model_with_tools = model.bind_tools(tools)

def agent_node(state: RAGAgentState):
    system = """คุณคือ "น้องเอ็กซ์" ผู้ช่วยของบริษัท XYZ
ใช้เครื่องมือที่เหมาะสมในการตอบ:
- search_hr_docs: สำหรับคำถามเรื่อง HR (ลา, สวัสดิการ, โบนัส)
- search_it_docs: สำหรับคำถามเรื่อง IT (ระบบ, รหัสผ่าน, VPN)
- calculate: สำหรับคำนวณตัวเลข
ตอบเป็นภาษาไทย สุภาพ"""

    messages = [{"role": "system", "content": system}] + state["messages"]
    return {"messages": [model_with_tools.invoke(messages)]}

# Build Graph
builder = StateGraph(RAGAgentState)
builder.add_node("agent", agent_node)
builder.add_node("tools", ToolNode(tools=tools))
builder.add_edge(START, "agent")
builder.add_conditional_edges("agent", tools_condition)
builder.add_edge("tools", "agent")

rag_agent = builder.compile()

# ทดสอบ
r = rag_agent.invoke({
    "messages": [{"role": "user", "content": "ค่าอาหาร + ค่าเดินทาง ต่อเดือนเท่าไหร่?"}]
})
print(r["messages"][-1].content)
# Agent จะ:
# 1. เรียก search_hr_docs("ค่าอาหาร ค่าเดินทาง")
# 2. ได้: ค่าอาหาร 100 บาท/วัน, ค่าเดินทาง 2,500 บาท/เดือน
# 3. เรียก calculate("100 * 22 + 2500")
# 4. ตอบ: "ค่าอาหาร 2,200 + ค่าเดินทาง 2,500 = 4,700 บาท/เดือน"
```

---

## 12. Evaluation

### 12.1 วัดคุณภาพ RAG ด้วย 3 มิติ

```
                    ┌──────────────────┐
                    │    User Query    │
                    └────────┬─────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
            ▼                ▼                ▼
    ┌──────────────┐  ┌────────────┐  ┌──────────────┐
    │  Retrieval   │  │ Generation │  │  End-to-End  │
    │  Relevance   │  │  Quality   │  │  Correctness │
    │              │  │            │  │              │
    │ "ดึง chunks  │  │ "LLM ตอบ  │  │ "คำตอบสุดท้าย│
    │  ที่เกี่ยวข้อง│  │  ได้ดีจาก  │  │  ถูกต้อง     │
    │  มาได้ไหม?"  │  │  context?" │  │  ไหม?"       │
    └──────────────┘  └────────────┘  └──────────────┘
```

### 12.2 ตัวอย่าง Evaluation

```python
from langsmith import evaluate, Client

client = Client()

# สร้าง Dataset
dataset = client.create_dataset("rag-eval-test")
test_cases = [
    {
        "inputs": {"question": "ลาพักร้อนได้กี่วัน?"},
        "outputs": {
            "answer": "15 วัน",
            "expected_source": "hr-policy.pdf"
        }
    },
    {
        "inputs": {"question": "VPN server คืออะไร?"},
        "outputs": {
            "answer": "vpn.xyz.com",
            "expected_source": "it-manual.pdf"
        }
    },
    {
        "inputs": {"question": "ค่าอาหารวันละเท่าไหร่?"},
        "outputs": {
            "answer": "100 บาท",
            "expected_source": "benefits.pdf"
        }
    },
]

for tc in test_cases:
    client.create_example(inputs=tc["inputs"], outputs=tc["outputs"], dataset_id=dataset.id)

# Target function
def rag_bot(inputs: dict) -> dict:
    result = ask(inputs["question"])
    return {
        "answer": result["answer"],
        "sources": result["sources"],
    }

# Evaluators
def answer_correctness(run, example) -> dict:
    """ตรวจว่าคำตอบมีข้อมูลที่ถูกต้องไหม"""
    predicted = run.outputs["answer"].lower()
    expected = example.outputs["answer"].lower()
    return {
        "key": "correctness",
        "score": 1.0 if expected in predicted else 0.0
    }

def source_correctness(run, example) -> dict:
    """ตรวจว่าอ้างอิงเอกสารถูกต้องไหม"""
    sources = run.outputs.get("sources", [])
    expected_source = example.outputs.get("expected_source", "")
    return {
        "key": "source_accuracy",
        "score": 1.0 if expected_source in sources else 0.0
    }

def no_hallucination(run, example) -> dict:
    """ใช้ LLM ตรวจว่ามี hallucination ไหม"""
    question = example.inputs["question"]
    answer = run.outputs["answer"]

    judge_response = model.invoke(f"""ตรวจว่าคำตอบนี้มีข้อมูลที่แต่งขึ้นไหม:
คำถาม: {question}
คำตอบ: {answer}
ตอบ: SAFE (ไม่ hallucinate) หรือ HALLUCINATION (แต่งข้อมูล)""")

    return {
        "key": "no_hallucination",
        "score": 1.0 if "SAFE" in judge_response.content else 0.0
    }

# Run Evaluation
results = evaluate(
    rag_bot,
    data="rag-eval-test",
    evaluators=[answer_correctness, source_correctness, no_hallucination],
    experiment_prefix="rag-v1"
)
```

---

## 13. Best Practices

### Chunking
```
✅ ทดลอง chunk_size หลายค่า (300, 500, 800) แล้ววัดผล
✅ ใส่ chunk_overlap เสมอ (10-20% ของ chunk_size)
✅ ใช้ splitter ที่เหมาะกับประเภทเอกสาร
✅ เก็บ metadata ที่มีประโยชน์ (source, date, category)
❌ อย่าตัดเล็กเกินไป (ขาดบริบท) หรือใหญ่เกินไป (retrieval ไม่แม่น)
```

### Retrieval
```
✅ ใช้ MMR แทน similarity (ได้ผลลัพธ์หลากหลายขึ้น)
✅ ใช้ metadata filter เพื่อ scope การค้นหา
✅ ลอง Hybrid Search (keyword + semantic)
✅ ลอง Multi-query สำหรับคำถามซับซ้อน
❌ อย่าดึง chunks น้อยเกินไป (k=1) หรือมากเกินไป (k=20)
```

### Generation
```
✅ บอก LLM ให้ตอบจาก context เท่านั้น
✅ บอก LLM ให้ปฏิเสธถ้าไม่มีข้อมูล
✅ ให้ LLM อ้างอิง source
✅ ใช้ temperature=0 สำหรับ factual Q&A
❌ อย่าให้ LLM "แต่งเพิ่ม" จาก knowledge ของตัวเอง
```

### Production
```
✅ ใช้ persistent vector store (ไม่ใช่ in-memory)
✅ อัปเดตข้อมูลเป็นระยะ (incremental indexing)
✅ Monitor retrieval quality ด้วย LangSmith
✅ สร้าง evaluation dataset และรัน eval สม่ำเสมอ
✅ Cache embeddings ที่ใช้บ่อย
✅ ตั้ง rate limit สำหรับ embedding API
```
