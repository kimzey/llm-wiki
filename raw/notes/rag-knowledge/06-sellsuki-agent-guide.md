# Sellsuki RAG Agent - Complete Implementation Guide

## สารบัญ

1. [ภาพรวมของระบบ](#1-ภาพรวมของระบบ)
2. [Phase 1: เตรียม Data](#2-phase-1-เตรียม-data)
3. [Phase 2: Chunking & Embedding](#3-phase-2-chunking--embedding)
4. [Phase 3: Vector Database (pgvector)](#4-phase-3-vector-database-pgvector)
5. [Phase 4: RAG Client Frameworks เปรียบเทียบ](#5-phase-4-rag-client-frameworks)
6. [Phase 5: สร้าง Agent](#6-phase-5-สร้าง-agent)
7. [Phase 6: Deployment](#7-phase-6-deployment)
8. [Implementation Plan](#8-implementation-plan)

---

## 1. ภาพรวมของระบบ

```
┌─────────────┐    ┌──────────────┐    ┌──────────────┐    ┌─────────┐
│  Company     │───▶│  Chunking &  │───▶│  pgvector     │───▶│  RAG    │
│  Documents   │    │  Embedding   │    │  (PostgreSQL) │    │  Agent  │
└─────────────┘    └──────────────┘    └──────────────┘    └─────────┘
                                                                │
     ┌──────────────────────────────────────────────────────────┘
     ▼
┌─────────────────────────────────────────────┐
│  User ถาม → Agent ค้น Vector DB → LLM ตอบ   │
└─────────────────────────────────────────────┘
```

**Flow ทั้งหมด:**

1. รวบรวมข้อมูลบริษัท (docs, wiki, FAQ, policy ฯลฯ)
2. หั่นข้อมูลเป็น chunks เล็กๆ
3. แปลง chunks เป็น vector embeddings
4. เก็บใน pgvector (PostgreSQL extension)
5. เมื่อ user ถาม → แปลงคำถามเป็น vector → ค้นหา chunks ที่เกี่ยวข้อง → ส่งให้ LLM ตอบ

---

## 2. Phase 1: เตรียม Data

### 2.1 ประเภทข้อมูลที่ต้องรวบรวม

| ประเภท | ตัวอย่าง | Format ที่พบบ่อย |
|--------|----------|-----------------|
| Company Policy | ระเบียบบริษัท, สวัสดิการ, วันลา | PDF, Google Docs |
| Product Info | คู่มือ Sellsuki, feature docs | Markdown, HTML |
| FAQ | คำถามที่ถามบ่อย | Spreadsheet, JSON |
| Process & SOP | ขั้นตอนการทำงาน | PDF, Docs |
| HR Info | โครงสร้างองค์กร, ตำแหน่งงาน | Docs, Spreadsheet |
| Meeting Notes | สรุปประชุม | Docs, Notion |
| Knowledge Base | บทความภายใน, wiki | Markdown, HTML |

### 2.2 การ Extract ข้อมูลจาก Source ต่างๆ

```python
# === ติดตั้ง dependencies ===
# pip install pypdf2 python-docx beautifulsoup4 pandas notion-client

# --- Extract จาก PDF ---
from PyPDF2 import PdfReader

def extract_pdf(file_path: str) -> str:
    reader = PdfReader(file_path)
    text = ""
    for page in reader.pages:
        text += page.extract_text() + "\n"
    return text

# --- Extract จาก Word (.docx) ---
from docx import Document

def extract_docx(file_path: str) -> str:
    doc = Document(file_path)
    return "\n".join([para.text for para in doc.paragraphs])

# --- Extract จาก HTML ---
from bs4 import BeautifulSoup

def extract_html(file_path: str) -> str:
    with open(file_path, 'r', encoding='utf-8') as f:
        soup = BeautifulSoup(f.read(), 'html.parser')
    # ลบ script และ style tags
    for tag in soup(['script', 'style', 'nav', 'footer']):
        tag.decompose()
    return soup.get_text(separator="\n", strip=True)

# --- Extract จาก CSV/Excel (FAQ format) ---
import pandas as pd

def extract_faq_csv(file_path: str) -> list[dict]:
    df = pd.read_csv(file_path)
    faqs = []
    for _, row in df.iterrows():
        faqs.append({
            "question": row["question"],
            "answer": row["answer"],
            "category": row.get("category", "general")
        })
    return faqs

# --- Extract จาก Notion ---
from notion_client import Client

def extract_notion(token: str, database_id: str) -> list[str]:
    notion = Client(auth=token)
    results = notion.databases.query(database_id=database_id)
    pages = []
    for page in results["results"]:
        page_id = page["id"]
        blocks = notion.blocks.children.list(block_id=page_id)
        text = ""
        for block in blocks["results"]:
            if block["type"] == "paragraph":
                rich_texts = block["paragraph"]["rich_text"]
                text += "".join([rt["plain_text"] for rt in rich_texts]) + "\n"
        pages.append(text)
    return pages
```

### 2.3 Data Cleaning

```python
import re

def clean_text(text: str) -> str:
    # ลบ whitespace ซ้ำ
    text = re.sub(r'\n{3,}', '\n\n', text)
    text = re.sub(r' {2,}', ' ', text)
    # ลบ special characters ที่ไม่จำเป็น
    text = re.sub(r'[\x00-\x08\x0b\x0c\x0e-\x1f]', '', text)
    # trim
    text = text.strip()
    return text

def create_document(text: str, metadata: dict) -> dict:
    """สร้าง document object พร้อม metadata"""
    return {
        "content": clean_text(text),
        "metadata": {
            "source": metadata.get("source", "unknown"),
            "category": metadata.get("category", "general"),
            "department": metadata.get("department", ""),
            "last_updated": metadata.get("last_updated", ""),
            "title": metadata.get("title", ""),
            "doc_type": metadata.get("doc_type", "document"),
        }
    }
```

### 2.4 โครงสร้าง Data ก่อน Chunking

```python
# ตัวอย่าง document ที่เตรียมเสร็จแล้ว
documents = [
    {
        "content": "นโยบายการลา: พนักงานมีสิทธิ์ลาพักร้อน 10 วันต่อปี...",
        "metadata": {
            "source": "hr_policy_2024.pdf",
            "category": "hr",
            "department": "HR",
            "last_updated": "2024-01-15",
            "title": "นโยบายการลา",
            "doc_type": "policy"
        }
    },
    {
        "content": "วิธีสร้าง Order ใน Sellsuki: 1. เข้าเมนู Orders...",
        "metadata": {
            "source": "sellsuki_manual.md",
            "category": "product",
            "department": "Product",
            "last_updated": "2024-03-01",
            "title": "คู่มือการใช้งาน Sellsuki",
            "doc_type": "manual"
        }
    }
]
```

---

## 3. Phase 2: Chunking & Embedding

### 3.1 Chunking คืออะไร และทำไมต้องหั่น

**ปัญหา:** ข้อมูลบริษัททั้งหมดอาจมีหลายร้อยหน้า แต่ LLM มี context window จำกัด และ embedding model ทำงานได้ดีกับข้อความสั้นๆ

**Chunking = การหั่นเอกสารใหญ่เป็นชิ้นเล็กๆ** ที่ยังคงความหมายครบถ้วน

### 3.2 กลยุทธ์การ Chunking

```
┌────────────────────────────────────────────────────────────────┐
│                    Chunking Strategies                         │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Fixed Size        : หั่นตามจำนวน characters/tokens         │
│  2. Recursive         : หั่นตาม paragraph → sentence → word    │
│  3. Semantic          : หั่นตามความหมาย (ใช้ embedding)         │
│  4. Document-based    : หั่นตามโครงสร้าง (header, section)      │
│  5. Sentence Window   : ใช้ sentence เดียว + context รอบข้าง   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

**แนะนำสำหรับ Sellsuki Agent: ใช้ Recursive + Metadata**

### 3.3 Chunking Implementation

```python
# pip install langchain-text-splitters tiktoken

from langchain_text_splitters import RecursiveCharacterTextSplitter

# === Strategy 1: Recursive Character Splitting (แนะนำ) ===
def chunk_recursive(text: str, chunk_size=1000, chunk_overlap=200) -> list[str]:
    """
    หั่นข้อความแบบ recursive
    - chunk_size: ขนาด chunk (characters) — 500-1500 เหมาะสม
    - chunk_overlap: ส่วนที่ซ้อนกันระหว่าง chunk เพื่อไม่ให้ขาดบริบท
    
    วิธีทำงาน:
    1. พยายามหั่นที่ "\n\n" (paragraph) ก่อน
    2. ถ้ายังใหญ่เกิน หั่นที่ "\n" (line break)
    3. ถ้ายังใหญ่เกิน หั่นที่ " " (space)
    4. สุดท้ายหั่นที่ character
    """
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=chunk_overlap,
        separators=["\n\n", "\n", "。", ".", " ", ""],  # ใส่ "。" สำหรับภาษาไทย/ญี่ปุ่น
        length_function=len,
    )
    return splitter.split_text(text)


# === Strategy 2: Semantic Chunking (ขั้นสูงกว่า) ===
# pip install langchain-experimental langchain-openai
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

def chunk_semantic(text: str) -> list[str]:
    """
    หั่นตามความหมาย — ถ้า embedding ของประโยคถัดไปต่างจากประโยคก่อนหน้ามาก
    จะตัดเป็น chunk ใหม่
    """
    embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
    splitter = SemanticChunker(
        embeddings,
        breakpoint_threshold_type="percentile",
        breakpoint_threshold_amount=75,
    )
    return splitter.split_text(text)


# === Strategy 3: Document-based (สำหรับ Markdown/HTML ที่มี headers) ===
from langchain_text_splitters import MarkdownHeaderTextSplitter

def chunk_markdown(text: str) -> list[dict]:
    """หั่นตาม header structure ของ Markdown"""
    headers_to_split = [
        ("#", "header_1"),
        ("##", "header_2"),
        ("###", "header_3"),
    ]
    splitter = MarkdownHeaderTextSplitter(headers_to_split)
    chunks = splitter.split_text(text)
    return [
        {
            "content": chunk.page_content,
            "metadata": chunk.metadata  # จะมี header_1, header_2 ติดมาด้วย
        }
        for chunk in chunks
    ]
```

### 3.4 ตัวอย่างการหั่นจริง

```python
# สมมติมีเอกสาร:
document = """
# นโยบายการลา

## ลาพักร้อน
พนักงานประจำมีสิทธิ์ลาพักร้อนปีละ 10 วัน โดยต้องแจ้งล่วงหน้า
อย่างน้อย 3 วันทำการ การลาพักร้อนต้องได้รับอนุมัติจากหัวหน้างาน
โดยตรง หากลาเกิน 3 วันติดต่อกันต้องแจ้งล่วงหน้า 1 สัปดาห์

## ลาป่วย
พนักงานมีสิทธิ์ลาป่วยปีละ 30 วัน โดยได้รับค่าจ้าง
หากลาป่วยเกิน 3 วันติดต่อกัน ต้องมีใบรับรองแพทย์
"""

# ผลลัพธ์จาก chunk_markdown:
# Chunk 1: {
#   "content": "พนักงานประจำมีสิทธิ์ลาพักร้อนปีละ 10 วัน...",
#   "metadata": {"header_1": "นโยบายการลา", "header_2": "ลาพักร้อน"}
# }
# Chunk 2: {
#   "content": "พนักงานมีสิทธิ์ลาป่วยปีละ 30 วัน...",
#   "metadata": {"header_1": "นโยบายการลา", "header_2": "ลาป่วย"}
# }
```

### 3.5 เลือก Chunk Size อย่างไร

```
Chunk Size เล็ก (200-500 chars):
  ✅ ค้นหาแม่นยำกว่า (precision สูง)
  ✅ เหมาะกับ FAQ, ข้อมูลสั้นๆ
  ❌ อาจขาดบริบท
  
Chunk Size กลาง (500-1000 chars):  ← แนะนำเริ่มต้น
  ✅ สมดุลระหว่าง precision และ context
  ✅ เหมาะกับเอกสารทั่วไป

Chunk Size ใหญ่ (1000-2000 chars):
  ✅ บริบทครบถ้วน
  ✅ เหมาะกับเอกสาร technical ยาวๆ
  ❌ ค้นหาอาจไม่แม่นยำ (noise เยอะ)

Overlap (100-200 chars):
  → ช่วยไม่ให้ข้อมูลหายตรงรอยต่อ chunk
```

### 3.6 Embedding

```python
# === Embedding คืออะไร ===
# การแปลงข้อความเป็น vector (array ของตัวเลข)
# เช่น "ลาพักร้อน" → [0.023, -0.145, 0.089, ..., 0.034]  (1536 มิติ)
# 
# ข้อความที่ความหมายคล้ายกัน → vector ที่อยู่ใกล้กัน
# "ลาพักร้อน" ≈ "หยุดพักผ่อน" (cosine similarity สูง)

# === ตัวเลือก Embedding Models ===
# 
# | Model                        | มิติ  | ราคา          | คุณภาพ |
# |------------------------------|-------|---------------|--------|
# | text-embedding-3-small       | 1536  | $0.02/1M tok  | ดี     |
# | text-embedding-3-large       | 3072  | $0.13/1M tok  | ดีมาก  |
# | text-embedding-ada-002       | 1536  | $0.10/1M tok  | ดี     |
# | Cohere embed-multilingual-v3 | 1024  | $0.10/1M tok  | ดีมาก  |
# | Google text-embedding-004    | 768   | Free tier มี  | ดี     |
# | BGE-M3 (open source)         | 1024  | ฟรี (self-host)| ดี    |

# แนะนำ: text-embedding-3-small (ราคาถูก คุณภาพดี)
# สำหรับภาษาไทย: Cohere embed-multilingual-v3 ดีมาก

import openai

client = openai.OpenAI(api_key="sk-xxx")

def get_embedding(text: str, model="text-embedding-3-small") -> list[float]:
    """แปลงข้อความเป็น vector"""
    response = client.embeddings.create(
        input=text,
        model=model
    )
    return response.data[0].embedding  # [0.023, -0.145, ...] (1536 ตัว)

def get_embeddings_batch(texts: list[str], model="text-embedding-3-small") -> list[list[float]]:
    """แปลงหลายข้อความพร้อมกัน (ประหยัด API calls)"""
    response = client.embeddings.create(
        input=texts,
        model=model
    )
    return [item.embedding for item in response.data]
```

### 3.7 Pipeline รวม: Document → Chunks → Embeddings

```python
import json
from pathlib import Path

def process_document(doc: dict, chunk_size=800, chunk_overlap=150) -> list[dict]:
    """แปลง document เป็น chunks พร้อม embeddings"""
    
    # 1. หั่น
    chunks = chunk_recursive(doc["content"], chunk_size, chunk_overlap)
    
    # 2. สร้าง embedding ทีเดียว (batch)
    embeddings = get_embeddings_batch(chunks)
    
    # 3. รวมผลลัพธ์
    results = []
    for i, (chunk, embedding) in enumerate(zip(chunks, embeddings)):
        results.append({
            "content": chunk,
            "embedding": embedding,
            "metadata": {
                **doc["metadata"],
                "chunk_index": i,
                "chunk_total": len(chunks),
            }
        })
    
    return results

# === รัน Pipeline ทั้งหมด ===
all_chunks = []
for doc in documents:
    chunks = process_document(doc)
    all_chunks.extend(chunks)
    print(f"Processed: {doc['metadata']['title']} → {len(chunks)} chunks")

print(f"\nTotal chunks: {len(all_chunks)}")
```

---

## 4. Phase 3: Vector Database (pgvector)

### 4.1 ทำไมเลือก pgvector

```
pgvector (PostgreSQL Extension):
  ✅ ใช้ PostgreSQL ที่คุ้นเคยอยู่แล้ว ไม่ต้องเรียนรู้ DB ใหม่
  ✅ SQL query ปกติ + vector search รวมกันได้
  ✅ Filter metadata ด้วย WHERE clause ได้เลย
  ✅ ไม่ต้อง deploy service แยก
  ✅ รองรับ index: IVFFlat, HNSW
  ✅ ฟรี, open source

เทียบกับตัวเลือกอื่น:
  Pinecone   → Managed service, ง่าย แต่มีค่าใช้จ่าย
  Weaviate   → Feature เยอะ แต่ต้อง deploy แยก
  ChromaDB   → ง่ายมาก เหมาะ prototype แต่ไม่เหมาะ production
  Qdrant     → Performance ดี แต่ต้อง deploy แยก
  Milvus     → Scale ได้มาก แต่ซับซ้อน
```

### 4.2 Setup pgvector

```sql
-- === ติดตั้ง pgvector Extension ===
-- PostgreSQL 14+ required
CREATE EXTENSION IF NOT EXISTS vector;

-- === สร้างตาราง ===
CREATE TABLE documents (
    id BIGSERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    embedding vector(1536),  -- 1536 มิติสำหรับ OpenAI embedding
    
    -- Metadata columns (แยกออกมาเพื่อ filter ได้เร็ว)
    source VARCHAR(500),
    category VARCHAR(100),
    department VARCHAR(100),
    doc_type VARCHAR(50),
    title VARCHAR(500),
    last_updated TIMESTAMP,
    chunk_index INTEGER,
    chunk_total INTEGER,
    
    -- Metadata เพิ่มเติม (flexible)
    metadata JSONB DEFAULT '{}',
    
    created_at TIMESTAMP DEFAULT NOW()
);

-- === สร้าง Index สำหรับ Vector Search ===

-- Option 1: IVFFlat Index (เร็วกว่าแต่ต้อง train)
-- เหมาะเมื่อมี data > 10,000 rows
-- lists = sqrt(จำนวน rows) เช่น data 10,000 rows → lists = 100
CREATE INDEX ON documents 
USING ivfflat (embedding vector_cosine_ops) 
WITH (lists = 100);

-- Option 2: HNSW Index (แนะนำ — แม่นยำกว่า ไม่ต้อง train)
-- m = จำนวน connections per node (16 default)
-- ef_construction = search width ตอนสร้าง index (64 default)
CREATE INDEX ON documents 
USING hnsw (embedding vector_cosine_ops) 
WITH (m = 16, ef_construction = 64);

-- === Index สำหรับ metadata filtering ===
CREATE INDEX idx_documents_category ON documents(category);
CREATE INDEX idx_documents_department ON documents(department);
CREATE INDEX idx_documents_doc_type ON documents(doc_type);
CREATE INDEX idx_documents_metadata ON documents USING gin(metadata);
```

### 4.3 อธิบาย Index แต่ละแบบ

```
IVFFlat (Inverted File with Flat Compression):
┌─────────────────────────────────────────────┐
│  แบ่ง vectors ออกเป็นกลุ่ม (lists/clusters)  │
│  ตอนค้นหา จะค้นแค่กลุ่มที่ใกล้ที่สุด         │
│                                              │
│  Pros: สร้าง index เร็ว, ใช้ memory น้อย      │
│  Cons: ต้อง rebuild เมื่อเพิ่ม data เยอะ      │
│        ความแม่นยำขึ้นกับจำนวน probes          │
│                                              │
│  เหมาะ: data ไม่ update บ่อย, > 10K rows     │
└─────────────────────────────────────────────┘

HNSW (Hierarchical Navigable Small World):
┌─────────────────────────────────────────────┐
│  สร้างกราฟหลายชั้น เชื่อม vectors ที่ใกล้กัน │
│  ค้นหาจากชั้นบนสุด (กว้าง) ลงล่างสุด (ละเอียด)│
│                                              │
│  Pros: แม่นยำมาก, ไม่ต้อง train              │
│        เพิ่ม data ได้เรื่อยๆ                   │
│  Cons: ใช้ memory มากกว่า, สร้าง index นานกว่า │
│                                              │
│  เหมาะ: ต้องการความแม่นยำสูง ← แนะนำ         │
└─────────────────────────────────────────────┘
```

### 4.4 Insert Data เข้า pgvector

```python
import psycopg2
from psycopg2.extras import execute_values
import numpy as np

# === Connection ===
conn = psycopg2.connect(
    host="localhost",
    port=5432,
    dbname="sellsuki_agent",
    user="postgres",
    password="your_password"
)

def insert_chunks(chunks: list[dict]):
    """Insert chunks พร้อม embeddings เข้า pgvector"""
    cur = conn.cursor()
    
    values = []
    for chunk in chunks:
        meta = chunk["metadata"]
        values.append((
            chunk["content"],
            chunk["embedding"],      # list[float] → pgvector แปลงอัตโนมัติ
            meta.get("source", ""),
            meta.get("category", ""),
            meta.get("department", ""),
            meta.get("doc_type", ""),
            meta.get("title", ""),
            meta.get("last_updated"),
            meta.get("chunk_index", 0),
            meta.get("chunk_total", 0),
            json.dumps(meta),        # เก็บ metadata ทั้งหมดใน JSONB ด้วย
        ))
    
    execute_values(
        cur,
        """
        INSERT INTO documents 
            (content, embedding, source, category, department, 
             doc_type, title, last_updated, chunk_index, chunk_total, metadata)
        VALUES %s
        """,
        values,
        template="""(
            %s, %s::vector, %s, %s, %s, 
            %s, %s, %s::timestamp, %s, %s, %s::jsonb
        )"""
    )
    
    conn.commit()
    cur.close()
    print(f"Inserted {len(chunks)} chunks")

# === รัน ===
insert_chunks(all_chunks)
```

### 4.5 Vector Search Queries

```sql
-- === Basic: ค้นหา 5 chunks ที่ใกล้เคียงที่สุด ===
-- ใช้ cosine distance (<=>)
SELECT 
    id,
    content,
    title,
    category,
    1 - (embedding <=> '[0.023, -0.145, ...]'::vector) AS similarity
FROM documents
ORDER BY embedding <=> '[0.023, -0.145, ...]'::vector
LIMIT 5;

-- === Filter + Vector Search ===
-- ค้นเฉพาะหมวด HR
SELECT 
    id,
    content,
    title,
    1 - (embedding <=> '[...]'::vector) AS similarity
FROM documents
WHERE category = 'hr'
ORDER BY embedding <=> '[...]'::vector
LIMIT 5;

-- === Filter หลายเงื่อนไข ===
SELECT 
    id,
    content,
    title,
    1 - (embedding <=> '[...]'::vector) AS similarity
FROM documents
WHERE category IN ('hr', 'policy')
  AND department = 'HR'
  AND last_updated > '2024-01-01'
ORDER BY embedding <=> '[...]'::vector
LIMIT 5;

-- === JSONB metadata query ===
SELECT content, title
FROM documents
WHERE metadata @> '{"doc_type": "faq"}'
ORDER BY embedding <=> '[...]'::vector
LIMIT 5;
```

### 4.6 Python Search Function

```python
def search_similar(
    query: str, 
    top_k: int = 5, 
    category: str = None,
    similarity_threshold: float = 0.7
) -> list[dict]:
    """ค้นหา chunks ที่เกี่ยวข้องกับ query"""
    
    # 1. แปลงคำถามเป็น embedding
    query_embedding = get_embedding(query)
    
    # 2. สร้าง SQL query
    cur = conn.cursor()
    
    sql = """
        SELECT 
            id,
            content,
            title,
            category,
            source,
            1 - (embedding <=> %s::vector) AS similarity
        FROM documents
        WHERE 1=1
    """
    params = [str(query_embedding)]
    
    if category:
        sql += " AND category = %s"
        params.append(category)
    
    sql += """
        ORDER BY embedding <=> %s::vector
        LIMIT %s
    """
    params.extend([str(query_embedding), top_k])
    
    cur.execute(sql, params)
    rows = cur.fetchall()
    cur.close()
    
    # 3. Filter by similarity threshold
    results = []
    for row in rows:
        if row[5] >= similarity_threshold:
            results.append({
                "id": row[0],
                "content": row[1],
                "title": row[2],
                "category": row[3],
                "source": row[4],
                "similarity": round(row[5], 4),
            })
    
    return results
```

### 4.7 Distance Functions ที่ใช้ได้

```
pgvector รองรับ 3 แบบ:

1. Cosine Distance (<=>)  ← แนะนำ
   - วัดมุมระหว่าง vectors
   - ไม่สนขนาด vector สนแค่ทิศทาง
   - ค่า 0 = เหมือนกัน, 2 = ตรงข้าม
   - similarity = 1 - distance

2. L2 Distance (<->)
   - Euclidean distance ปกติ
   - สนทั้งทิศทางและขนาด
   - ค่ายิ่งน้อยยิ่งใกล้

3. Inner Product (<#>)
   - Dot product (negative)
   - เร็วที่สุด
   - ใช้ได้ดีเมื่อ vectors ถูก normalize แล้ว
```

---

## 5. Phase 4: RAG Client Frameworks

### 5.1 เปรียบเทียบ Frameworks ทั้งหมด

```
┌────────────────────┬──────────┬──────────┬──────────┬──────────┐
│                    │LangChain │Google ADK│LlamaIndex│ CrewAI   │
├────────────────────┼──────────┼──────────┼──────────┼──────────┤
│ ความง่าย           │ ★★★☆☆   │ ★★★★☆   │ ★★★★☆   │ ★★★★☆   │
│ Flexibility        │ ★★★★★   │ ★★★☆☆   │ ★★★★☆   │ ★★★☆☆   │
│ RAG Support        │ ★★★★★   │ ★★★☆☆   │ ★★★★★   │ ★★★☆☆   │
│ Agent Support      │ ★★★★★   │ ★★★★★   │ ★★★★☆   │ ★★★★★   │
│ Community          │ ★★★★★   │ ★★★☆☆   │ ★★★★☆   │ ★★★☆☆   │
│ Production Ready   │ ★★★★☆   │ ★★★☆☆   │ ★★★★☆   │ ★★★☆☆   │
│ Multi-LLM          │ ★★★★★   │ ★★★☆☆   │ ★★★★★   │ ★★★★☆   │
│ Thai Language      │ ★★★★☆   │ ★★★★☆   │ ★★★★☆   │ ★★★★☆   │
└────────────────────┴──────────┴──────────┴──────────┴──────────┘

อื่นๆ ที่น่าสนใจ:
- Haystack (deepset): เน้น RAG โดยเฉพาะ, production-ready
- AutoGen (Microsoft): Multi-agent conversations
- Semantic Kernel (Microsoft): Enterprise-grade, C#/.NET friendly
- Vercel AI SDK: สำหรับ Next.js/TypeScript projects
- Dify: Low-code RAG platform (มี UI)
- DIY (ไม่ใช้ framework): ควบคุมได้ 100%, เหมาะ simple use case
```

### 5.2 LangChain Implementation

```python
# pip install langchain langchain-openai langchain-postgres

from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_postgres import PGVector
from langchain.chains import RetrievalQA
from langchain.prompts import PromptTemplate
from langchain.agents import AgentExecutor, create_openai_functions_agent
from langchain.tools import Tool

# === 1. Setup Vector Store ===
CONNECTION_STRING = "postgresql+psycopg://postgres:password@localhost:5432/sellsuki_agent"

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

vectorstore = PGVector(
    embeddings=embeddings,
    collection_name="sellsuki_docs",
    connection=CONNECTION_STRING,
    use_jsonb=True,
)

# === 2. สร้าง Retriever ===
retriever = vectorstore.as_retriever(
    search_type="similarity_score_threshold",
    search_kwargs={
        "k": 5,                        # จำนวน chunks ที่ดึงมา
        "score_threshold": 0.7,         # minimum similarity
    }
)

# === 3. Custom Prompt สำหรับ Sellsuki ===
SELLSUKI_PROMPT = PromptTemplate(
    template="""คุณเป็น AI Assistant ของบริษัท Sellsuki 
ชื่อว่า "Suki Bot" ทำหน้าที่ตอบคำถามเกี่ยวกับบริษัท

กฎ:
1. ตอบเฉพาะข้อมูลที่มีใน context เท่านั้น
2. ถ้าไม่มีข้อมูล ให้บอกว่า "ขออภัย ไม่พบข้อมูลเกี่ยวกับเรื่องนี้ กรุณาติดต่อ HR/Admin"
3. ตอบเป็นภาษาไทย สุภาพ เป็นกันเอง
4. ถ้ามีหลายข้อมูลที่เกี่ยวข้อง ให้สรุปรวมให้ครบ
5. อ้างอิงแหล่งที่มาของข้อมูลเสมอ

Context:
{context}

คำถาม: {question}

คำตอบ:""",
    input_variables=["context", "question"]
)

# === 4. สร้าง RAG Chain ===
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    chain_type="stuff",          # stuff = ยัด context ทั้งหมดใน prompt เดียว
    retriever=retriever,
    chain_type_kwargs={"prompt": SELLSUKI_PROMPT},
    return_source_documents=True,
)

# === 5. ใช้งาน ===
result = qa_chain.invoke({"query": "วันลาพักร้อนมีกี่วัน?"})
print(result["result"])
# → "พนักงานประจำมีสิทธิ์ลาพักร้อนปีละ 10 วัน (อ้างอิง: นโยบายการลา)"

# === 6. สร้าง Agent (ขั้นสูง — มี tools หลายตัว) ===
from langchain.tools.retriever import create_retriever_tool

# Tool 1: ค้นหาข้อมูลทั่วไป
general_tool = create_retriever_tool(
    retriever,
    "search_company_info",
    "ค้นหาข้อมูลทั่วไปของบริษัท Sellsuki เช่น นโยบาย สวัสดิการ วิธีการทำงาน"
)

# Tool 2: ค้นหาข้อมูล HR โดยเฉพาะ
hr_retriever = vectorstore.as_retriever(
    search_kwargs={"k": 3, "filter": {"category": "hr"}}
)
hr_tool = create_retriever_tool(
    hr_retriever,
    "search_hr_info",
    "ค้นหาข้อมูลด้าน HR เช่น การลา สวัสดิการ เงินเดือน"
)

from langchain.prompts import ChatPromptTemplate, MessagesPlaceholder

agent_prompt = ChatPromptTemplate.from_messages([
    ("system", """คุณเป็น Suki Bot — AI Assistant ของ Sellsuki
    ใช้ tools ค้นหาข้อมูลก่อนตอบเสมอ ห้ามเดาคำตอบ"""),
    MessagesPlaceholder(variable_name="chat_history", optional=True),
    ("human", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad"),
])

agent = create_openai_functions_agent(llm, [general_tool, hr_tool], agent_prompt)
agent_executor = AgentExecutor(agent=agent, tools=[general_tool, hr_tool], verbose=True)

response = agent_executor.invoke({"input": "ลาป่วยต้องมีใบรับรองแพทย์มั้ย?"})
print(response["output"])
```

### 5.3 Google ADK (Agent Development Kit)

```python
# pip install google-adk google-genai

from google import genai
from google.adk.agents import Agent
from google.adk.tools import FunctionTool
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService

# === 1. สร้าง Tool สำหรับค้น Vector DB ===
def search_company_knowledge(query: str, category: str = "") -> str:
    """ค้นหาข้อมูลบริษัท Sellsuki จาก knowledge base
    
    Args:
        query: คำถามหรือคีย์เวิร์ดที่ต้องการค้นหา
        category: หมวดหมู่ เช่น hr, product, policy (ถ้าไม่ระบุจะค้นทั้งหมด)
    
    Returns:
        ข้อมูลที่เกี่ยวข้องจาก knowledge base
    """
    results = search_similar(query, top_k=5, category=category or None)
    
    if not results:
        return "ไม่พบข้อมูลที่เกี่ยวข้อง"
    
    context = ""
    for i, r in enumerate(results, 1):
        context += f"\n[{i}] ({r['title']}) {r['content']}\n"
    
    return context

# === 2. สร้าง Agent ===
sellsuki_agent = Agent(
    name="suki_bot",
    model="gemini-2.0-flash",
    description="AI Assistant สำหรับตอบคำถามเกี่ยวกับบริษัท Sellsuki",
    instruction="""คุณเป็น Suki Bot — AI Assistant ของบริษัท Sellsuki
    
    กฎการทำงาน:
    1. ใช้ tool search_company_knowledge ค้นหาข้อมูลก่อนตอบทุกครั้ง
    2. ตอบเฉพาะข้อมูลที่ค้นพบเท่านั้น ห้ามเดา
    3. ตอบเป็นภาษาไทย สุภาพ เข้าใจง่าย
    4. อ้างอิงแหล่งที่มาเสมอ
    5. ถ้าไม่พบข้อมูล ให้แนะนำติดต่อ HR หรือ Admin
    """,
    tools=[search_company_knowledge],
)

# === 3. Runner & Session ===
session_service = InMemorySessionService()
runner = Runner(
    agent=sellsuki_agent,
    app_name="sellsuki_assistant",
    session_service=session_service,
)

# === 4. ใช้งาน ===
import asyncio
from google.genai import types

async def chat(user_message: str, user_id: str = "user_1"):
    session = await session_service.create_session(
        app_name="sellsuki_assistant",
        user_id=user_id,
    )
    
    content = types.Content(
        role="user",
        parts=[types.Part.from_text(user_message)]
    )
    
    response_text = ""
    async for event in runner.run_async(
        user_id=user_id,
        session_id=session.id,
        new_message=content,
    ):
        if event.is_final_response():
            for part in event.content.parts:
                response_text += part.text
    
    return response_text

# รัน
result = asyncio.run(chat("สวัสดิการบริษัทมีอะไรบ้าง?"))
print(result)
```

### 5.4 LlamaIndex Implementation

```python
# pip install llama-index llama-index-vector-stores-postgres llama-index-embeddings-openai

from llama_index.core import (
    VectorStoreIndex,
    SimpleDirectoryReader,
    StorageContext,
    Settings,
)
from llama_index.vector_stores.postgres import PGVectorStore
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.llms.openai import OpenAI
import sqlalchemy

# === 1. Setup ===
Settings.llm = OpenAI(model="gpt-4o-mini", temperature=0)
Settings.embed_model = OpenAIEmbedding(model_name="text-embedding-3-small")

# === 2. Vector Store ===
vector_store = PGVectorStore.from_params(
    database="sellsuki_agent",
    host="localhost",
    password="your_password",
    port=5432,
    user="postgres",
    table_name="llama_docs",
    embed_dim=1536,
)

storage_context = StorageContext.from_defaults(vector_store=vector_store)

# === 3. Load Documents & Index ===
# อ่านไฟล์ทั้งหมดจากโฟลเดอร์
documents = SimpleDirectoryReader("./company_docs/").load_data()

# สร้าง index (chunk + embed + store อัตโนมัติ)
index = VectorStoreIndex.from_documents(
    documents,
    storage_context=storage_context,
    show_progress=True,
)

# === 4. Query ===
query_engine = index.as_query_engine(
    similarity_top_k=5,
    response_mode="compact",  # compact, tree_summarize, refine
)

response = query_engine.query("วิธีสร้าง order ใน Sellsuki ทำยังไง?")
print(response)

# === 5. Chat Engine (มี memory) ===
chat_engine = index.as_chat_engine(
    chat_mode="context",  # context, condense_question, condense_plus_context
    system_prompt="คุณเป็น Suki Bot ตอบคำถามเกี่ยวกับ Sellsuki เป็นภาษาไทย",
)

response = chat_engine.chat("ลาพักร้อนกี่วัน?")
print(response)

response = chat_engine.chat("แล้วลาป่วยล่ะ?")  # จำบริบทก่อนหน้าได้
print(response)
```

### 5.5 DIY (ไม่ใช้ Framework)

```python
# === วิธีทำเอง — ง่าย เข้าใจทุกส่วน ===
import openai

client = openai.OpenAI(api_key="sk-xxx")

SYSTEM_PROMPT = """คุณเป็น Suki Bot — AI Assistant ของบริษัท Sellsuki
ตอบคำถามเกี่ยวกับบริษัทโดยอ้างอิงจาก context ที่ให้มาเท่านั้น
ถ้าไม่มีข้อมูล ให้บอกว่าไม่พบ แนะนำติดต่อ HR/Admin
ตอบเป็นภาษาไทย สุภาพ"""

def ask_agent(question: str, chat_history: list = None) -> str:
    # 1. ค้นหาข้อมูลที่เกี่ยวข้อง
    results = search_similar(question, top_k=5)
    
    # 2. สร้าง context
    context = "\n\n".join([
        f"[{r['title']}]: {r['content']}" 
        for r in results
    ])
    
    # 3. สร้าง messages
    messages = [{"role": "system", "content": SYSTEM_PROMPT}]
    
    if chat_history:
        messages.extend(chat_history)
    
    messages.append({
        "role": "user",
        "content": f"""ข้อมูลอ้างอิง:
{context}

คำถาม: {question}"""
    })
    
    # 4. เรียก LLM
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=messages,
        temperature=0,
    )
    
    return response.choices[0].message.content

# ใช้งาน
print(ask_agent("สวัสดิการบริษัทมีอะไรบ้าง?"))
```

### 5.6 สรุปการเลือก Framework

```
ถ้าคุณ...

→ ต้องการ flexibility สูง, ใช้ tools เยอะ, multi-agent
  ใช้ LangChain

→ ต้องการ focus เรื่อง RAG, query เก่ง, document หลายแบบ  
  ใช้ LlamaIndex

→ ใช้ Google Cloud อยู่แล้ว, ต้องการ integrate Gemini
  ใช้ Google ADK

→ ต้องการ multi-agent ทำงานร่วมกัน
  ใช้ CrewAI หรือ AutoGen

→ ต้องการเข้าใจทุกส่วน, use case ไม่ซับซ้อน
  ใช้ DIY

→ ต้องการ low-code, มี UI จัดการ
  ใช้ Dify

สำหรับ Sellsuki Agent: แนะนำ LangChain หรือ LlamaIndex
เพราะ community ใหญ่ มี pgvector support ดี
```

---

## 6. Phase 5: สร้าง Agent แบบเต็ม

### 6.1 โครงสร้าง Project

```
sellsuki-agent/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI entry point
│   ├── config.py             # Configuration
│   ├── agent/
│   │   ├── __init__.py
│   │   ├── agent.py          # Agent logic
│   │   ├── prompts.py        # System prompts
│   │   └── tools.py          # Agent tools
│   ├── rag/
│   │   ├── __init__.py
│   │   ├── embeddings.py     # Embedding functions
│   │   ├── retriever.py      # Vector search
│   │   └── chunker.py        # Chunking logic
│   ├── db/
│   │   ├── __init__.py
│   │   ├── connection.py     # Database connection
│   │   └── models.py         # SQLAlchemy models
│   └── ingest/
│       ├── __init__.py
│       ├── pipeline.py       # Data ingestion pipeline
│       └── extractors.py     # Document extractors
├── scripts/
│   ├── ingest_data.py        # Script to load data
│   └── test_search.py        # Test vector search
├── data/
│   └── company_docs/         # Raw documents
├── tests/
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
└── .env
```

### 6.2 API Server (FastAPI)

```python
# app/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from app.agent.agent import SellsukiAgent

app = FastAPI(title="Sellsuki Agent API")
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])

agent = SellsukiAgent()

class ChatRequest(BaseModel):
    message: str
    session_id: str = "default"
    category: str | None = None

class ChatResponse(BaseModel):
    answer: str
    sources: list[dict]
    session_id: str

# In-memory chat history (production ใช้ Redis)
chat_histories: dict[str, list] = {}

@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    history = chat_histories.get(request.session_id, [])
    
    answer, sources = await agent.ask(
        question=request.message,
        chat_history=history,
        category=request.category,
    )
    
    # Update history
    history.append({"role": "user", "content": request.message})
    history.append({"role": "assistant", "content": answer})
    chat_histories[request.session_id] = history[-20:]  # keep last 10 turns
    
    return ChatResponse(
        answer=answer,
        sources=sources,
        session_id=request.session_id,
    )

@app.post("/ingest")
async def ingest_document(file_path: str, category: str = "general"):
    """Endpoint สำหรับเพิ่มเอกสารใหม่"""
    from app.ingest.pipeline import ingest_file
    count = await ingest_file(file_path, category)
    return {"status": "ok", "chunks_added": count}

@app.get("/health")
async def health():
    return {"status": "healthy"}
```

### 6.3 Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  # PostgreSQL + pgvector
  db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: sellsuki_agent
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql

  # Agent API
  agent:
    build: .
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://postgres:${DB_PASSWORD}@db:5432/sellsuki_agent
      OPENAI_API_KEY: ${OPENAI_API_KEY}
    depends_on:
      - db

  # Redis (สำหรับ session/cache)
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

---

## 7. Phase 6: Deployment

### 7.1 ตัวเลือก Deployment

```
┌─────────────────────────────────────────────────────────────────┐
│                    Deployment Options                           │
├───────────────┬─────────┬──────────┬──────────┬────────────────┤
│               │ ราคา     │ ความง่าย  │ Scale    │ เหมาะกับ       │
├───────────────┼─────────┼──────────┼──────────┼────────────────┤
│ Self-hosted   │ ถูก     │ ★★☆☆☆   │ Manual   │ ทีม DevOps มี   │
│ (VM/VPS)      │         │          │          │                │
├───────────────┼─────────┼──────────┼──────────┼────────────────┤
│ Docker +      │ ถูก-กลาง │ ★★★☆☆   │ Auto     │ มี K8s อยู่แล้ว │
│ Kubernetes    │         │          │          │                │
├───────────────┼─────────┼──────────┼──────────┼────────────────┤
│ Google        │ กลาง    │ ★★★★☆   │ Auto     │ ใช้ GCP อยู่    │
│ Cloud Run     │         │          │          │                │
├───────────────┼─────────┼──────────┼──────────┼────────────────┤
│ AWS ECS/      │ กลาง    │ ★★★☆☆   │ Auto     │ ใช้ AWS อยู่    │
│ Fargate       │         │          │          │                │
├───────────────┼─────────┼──────────┼──────────┼────────────────┤
│ Railway/      │ กลาง    │ ★★★★★   │ Auto     │ ต้องการความง่าย  │
│ Render        │         │          │          │                │
├───────────────┼─────────┼──────────┼──────────┼────────────────┤
│ Google Vertex │ สูง     │ ★★★★☆   │ Auto     │ ใช้ ADK + Gemini│
│ AI Agent      │         │          │          │                │
│ Engine        │         │          │          │                │
├───────────────┼─────────┼──────────┼──────────┼────────────────┤
│ LangServe/    │ ฟรี-สูง │ ★★★★☆   │ Varies   │ ใช้ LangChain   │
│ LangSmith     │         │          │          │                │
└───────────────┴─────────┴──────────┴──────────┴────────────────┘
```

### 7.2 Deploy ด้วย Google Cloud Run (แนะนำ)

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Cloud Run ใช้ PORT env variable
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "${PORT:-8000}"]
```

```bash
# === Deploy Steps ===

# 1. Setup Google Cloud
gcloud auth login
gcloud config set project YOUR_PROJECT_ID

# 2. สร้าง Cloud SQL (PostgreSQL + pgvector)
gcloud sql instances create sellsuki-db \
    --database-version=POSTGRES_15 \
    --tier=db-f1-micro \
    --region=asia-southeast1 \
    --database-flags=cloudsql.enable_pgvector=on

gcloud sql databases create sellsuki_agent \
    --instance=sellsuki-db

# 3. Build & Push Docker image
gcloud builds submit --tag gcr.io/YOUR_PROJECT/sellsuki-agent

# 4. Deploy to Cloud Run
gcloud run deploy sellsuki-agent \
    --image gcr.io/YOUR_PROJECT/sellsuki-agent \
    --platform managed \
    --region asia-southeast1 \
    --allow-unauthenticated \
    --add-cloudsql-instances YOUR_PROJECT:asia-southeast1:sellsuki-db \
    --set-env-vars "OPENAI_API_KEY=sk-xxx" \
    --set-env-vars "DATABASE_URL=postgresql://..." \
    --memory 1Gi \
    --cpu 1 \
    --min-instances 0 \
    --max-instances 5

# ได้ URL: https://sellsuki-agent-xxxx-as.a.run.app
```

### 7.3 Deploy ด้วย LangServe (สำหรับ LangChain)

```python
# pip install langserve[all]

# serve.py
from fastapi import FastAPI
from langserve import add_routes
from app.agent.agent import create_chain

app = FastAPI(title="Sellsuki Agent")

# เพิ่ม LangServe routes
add_routes(
    app,
    create_chain(),
    path="/chat",
)

# Deploy ได้เหมือน FastAPI ปกติ
```

### 7.4 Deploy Google ADK บน Vertex AI

```bash
# === Deploy ADK Agent บน Vertex AI Agent Engine ===

# 1. ติดตั้ง
pip install google-cloud-aiplatform[adk]

# 2. Deploy
from google.cloud import aiplatform
from google.adk.agents import Agent

aiplatform.init(project="your-project", location="us-central1")

# สร้าง agent (เหมือน section 5.3)
agent = Agent(name="suki_bot", ...)

# Deploy
remote_agent = aiplatform.Agent.create(
    agent=agent,
    display_name="Sellsuki Assistant",
    requirements=["psycopg2-binary", "openai"],
)

# ใช้งาน
session = remote_agent.create_session(user_id="user_1")
response = remote_agent.query(
    session_id=session.id,
    message="ลาพักร้อนกี่วัน?"
)
print(response.text)
```

### 7.5 Self-hosted (Docker Compose บน VPS)

```bash
# === Deploy บน VPS (เช่น DigitalOcean, Linode) ===

# 1. SSH เข้า server
ssh root@your-server-ip

# 2. ติดตั้ง Docker
curl -fsSL https://get.docker.com | sh

# 3. Clone project
git clone https://github.com/your-org/sellsuki-agent.git
cd sellsuki-agent

# 4. สร้าง .env
cat > .env << EOF
DB_PASSWORD=your_secure_password
OPENAI_API_KEY=sk-xxx
EOF

# 5. Run
docker compose up -d

# 6. Setup Nginx reverse proxy + SSL
apt install nginx certbot python3-certbot-nginx

# /etc/nginx/sites-available/agent
# server {
#     server_name agent.sellsuki.com;
#     location / {
#         proxy_pass http://localhost:8000;
#         proxy_set_header Host $host;
#     }
# }

certbot --nginx -d agent.sellsuki.com
```

---

## 8. Implementation Plan

### สรุป Timeline

```
Week 1-2: Phase 1 & 2 — เตรียม Data + Chunking
Week 3:   Phase 3     — Setup pgvector + Insert Data
Week 4:   Phase 4 & 5 — สร้าง Agent + API
Week 5:   Phase 6     — Deploy + Testing
Week 6:   Polish      — Fine-tune + UAT
```

### Week 1-2: Data Preparation

```
Day 1-2: รวบรวมเอกสารทั้งหมด
  □ รวม policy documents (PDF/Docs)
  □ รวม product documentation
  □ รวม FAQ จากทุก department
  □ รวม SOP / process documents
  □ ตรวจสอบว่าข้อมูล up-to-date

Day 3-5: Extract & Clean
  □ เขียน extractors สำหรับแต่ละ format
  □ Clean data (ลบ noise, format ให้เรียบร้อย)
  □ เพิ่ม metadata ให้ทุก document

Day 6-8: Chunking
  □ ทดลอง chunk size ต่างๆ (500, 800, 1000)
  □ เลือก strategy ที่เหมาะ (recursive vs semantic)
  □ ตรวจ chunks ว่ามีความหมายครบ

Day 9-10: Embedding
  □ เลือก embedding model
  □ Generate embeddings ทั้งหมด
  □ ตรวจ quality ด้วย sample queries
```

### Week 3: Database Setup

```
Day 11-12: Setup PostgreSQL + pgvector
  □ สร้าง database + tables
  □ สร้าง indexes (HNSW)
  □ Insert data ทั้งหมด

Day 13-15: Test & Optimize
  □ ทดสอบ search queries 50+ คำถาม
  □ ปรับ similarity threshold
  □ ปรับ top_k
  □ Benchmark performance
```

### Week 4: Agent Development

```
Day 16-17: เลือก Framework & Setup
  □ ตัดสินใจ: LangChain / LlamaIndex / DIY
  □ Setup project structure
  □ เขียน config

Day 18-19: สร้าง Agent Logic
  □ เขียน retriever
  □ เขียน agent + tools
  □ เขียน system prompt (สำคัญมาก!)
  □ เพิ่ม chat history

Day 20: สร้าง API
  □ FastAPI endpoints
  □ Session management
  □ Error handling
```

### Week 5: Deployment

```
Day 21-22: Containerize
  □ เขียน Dockerfile
  □ เขียน docker-compose.yml
  □ Test locally

Day 23-24: Deploy
  □ Setup cloud infrastructure
  □ Deploy database
  □ Deploy agent API
  □ Setup domain + SSL

Day 25: Integration
  □ เชื่อมกับ Slack / LINE / Web UI
  □ Test end-to-end
```

### Week 6: Polish & Launch

```
Day 26-27: Fine-tune
  □ ทดสอบกับ user จริง 10-20 คน
  □ ปรับ prompt ตาม feedback
  □ เพิ่มข้อมูลที่ขาด

Day 28-29: Monitoring & Logging
  □ Setup logging (query + response)
  □ Setup monitoring (uptime, latency)
  □ Setup alert

Day 30: Launch!
  □ ประกาศให้ทั้งบริษัทใช้
  □ เตรียม feedback channel
  □ Plan สำหรับ continuous improvement
```

### Cost Estimation (ต่อเดือน)

```
Embedding (text-embedding-3-small):
  - Ingest ครั้งแรก: ~$1-5 (ขึ้นกับจำนวน docs)
  - Query: ~$0.5-2/เดือน (1000 queries)

LLM (gpt-4o-mini):
  - ~$5-20/เดือน (1000 queries, avg 500 tokens/query)

Database:
  - Self-hosted: $5-10/เดือน (VPS)
  - Cloud SQL: $15-30/เดือน

Hosting:
  - Cloud Run: $5-15/เดือน (low traffic)
  - VPS: $5-10/เดือน

รวมประมาณ: $20-70/เดือน สำหรับ internal use
```

---

## Appendix: Quick Reference

### ติดตั้งทั้งหมดในคำสั่งเดียว

```bash
pip install \
    langchain langchain-openai langchain-postgres langchain-text-splitters \
    openai psycopg2-binary sqlalchemy \
    fastapi uvicorn python-multipart \
    pypdf2 python-docx beautifulsoup4 pandas \
    tiktoken
```

### ทดสอบว่าทุกอย่างทำงาน

```python
# test_everything.py
import openai
import psycopg2

# 1. Test OpenAI
client = openai.OpenAI()
emb = client.embeddings.create(input="test", model="text-embedding-3-small")
print(f"✅ OpenAI Embedding: {len(emb.data[0].embedding)} dimensions")

# 2. Test pgvector
conn = psycopg2.connect("postgresql://postgres:password@localhost/sellsuki_agent")
cur = conn.cursor()
cur.execute("SELECT 1")
print(f"✅ PostgreSQL connected")

cur.execute("CREATE EXTENSION IF NOT EXISTS vector")
conn.commit()
print(f"✅ pgvector extension ready")

# 3. Test vector search
cur.execute("SELECT '[1,2,3]'::vector <=> '[4,5,6]'::vector AS distance")
print(f"✅ Vector distance: {cur.fetchone()[0]}")

conn.close()
print("\n🎉 All systems ready!")
```
