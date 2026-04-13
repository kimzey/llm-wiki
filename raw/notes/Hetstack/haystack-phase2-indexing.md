# 🌾 Haystack Deep Dive — Phase 2: Document Processing & Indexing Pipeline

> **เป้าหมาย:** เข้าใจวิธีนำเข้าเอกสาร ประมวลผล และเก็บลงใน Document Store

---

## 1. ภาพรวม Indexing Pipeline

```
┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌───────────────┐
│  Files/  │ →  │  Converter   │ →  │ PreProcessor │ →  │ Document      │
│  Text    │    │  (แปลงไฟล์)   │    │  (ตัด chunk)  │    │ Store (เก็บ)  │
└──────────┘    └──────────────┘    └──────────────┘    └───────────────┘
                                                                ↑
                                              ┌─────────────────┘
                                              │  Embedder (ถ้าใช้ vector search)
                                              └──────────────────────────────
```

---

## 2. File Converters — แปลงไฟล์เป็น Document

Haystack รองรับการแปลงไฟล์หลายประเภท:

### 2.1 TextFileToDocument

```python
from haystack.components.converters import TextFileToDocument

converter = TextFileToDocument()
result = converter.run(sources=["article.txt", "readme.txt"])
documents = result["documents"]
```

### 2.2 PyPDFToDocument

```python
from haystack.components.converters import PyPDFToDocument

converter = PyPDFToDocument()
result = converter.run(sources=["report.pdf", "manual.pdf"])
documents = result["documents"]

# ติดตั้ง: pip install pypdf
```

### 2.3 DOCXToDocument

```python
from haystack.components.converters import DOCXToDocument

converter = DOCXToDocument()
result = converter.run(sources=["document.docx"])
documents = result["documents"]

# ติดตั้ง: pip install python-docx
```

### 2.4 HTMLToDocument

```python
from haystack.components.converters import HTMLToDocument

converter = HTMLToDocument()
result = converter.run(sources=["page.html"])
documents = result["documents"]

# ติดตั้ง: pip install boilerpy3
```

### 2.5 LinkContentFetcher — ดึงจาก URL

```python
from haystack.components.fetchers import LinkContentFetcher
from haystack.components.converters import HTMLToDocument

fetcher = LinkContentFetcher()
converter = HTMLToDocument()

# ดึง content จาก URL
fetch_result = fetcher.run(urls=["https://example.com/article"])
docs = converter.run(sources=fetch_result["streams"])
```

### 2.6 JSONConverter

```python
from haystack.components.converters import JSONConverter

# แปลง JSON เป็น Document โดยระบุ field ที่เป็น content
converter = JSONConverter(
    content_key="body",          # field ที่เป็นเนื้อหา
    meta_fields_to_embed=["title"]  # field ที่จะรวมใน metadata
)
result = converter.run(sources=["data.json"])
```

---

## 3. DocumentSplitter — ตัดเอกสารเป็น Chunk

เอกสารยาวต้องตัดเป็นชิ้นๆ เพราะ LLM มี context window จำกัด

```python
from haystack.components.preprocessors import DocumentSplitter

# ตัดตาม word count
splitter = DocumentSplitter(
    split_by="word",          # "word", "sentence", "passage", "page"
    split_length=200,         # ขนาดแต่ละ chunk (จำนวน word)
    split_overlap=20,         # overlap ระหว่าง chunk (กันข้อมูลหาย)
)

result = splitter.run(documents=documents)
chunks = result["documents"]
print(f"ตัดได้ {len(chunks)} chunks")
```

### ตัวเลือก split_by

| split_by | ความหมาย | เหมาะกับ |
|----------|---------|---------|
| `"word"` | ตามจำนวนคำ | เอกสารทั่วไป |
| `"sentence"` | ตามประโยค | ข้อความที่มีโครงสร้างชัด |
| `"passage"` | ตามย่อหน้า | บทความ, บล็อก |
| `"page"` | ตามหน้า | PDF หลายหน้า |

### Split Overlap คืออะไร?

```
ไม่มี overlap:
[chunk1: คำที่ 1-200] [chunk2: คำที่ 201-400] [chunk3: คำที่ 401-600]
                       ↑ ถ้าคำตอบอยู่ตรงรอยต่อ จะหายไป!

มี overlap=20:
[chunk1: คำที่ 1-200] 
             [chunk2: คำที่ 181-380]  ← overlap 20 คำ
                          [chunk3: คำที่ 361-560]
```

---

## 4. DocumentCleaner — ทำความสะอาดเอกสาร

```python
from haystack.components.preprocessors import DocumentCleaner

cleaner = DocumentCleaner(
    remove_empty_lines=True,           # ลบบรรทัดว่าง
    remove_extra_whitespaces=True,     # ลบ whitespace เกิน
    remove_repeated_substrings=False,  # ลบข้อความซ้ำ (เช่น header/footer)
)

result = cleaner.run(documents=raw_documents)
clean_docs = result["documents"]
```

---

## 5. Document Stores อย่างละเอียด

### 5.1 InMemoryDocumentStore (สำหรับ Dev/Test)

```python
from haystack.document_stores.in_memory import InMemoryDocumentStore

store = InMemoryDocumentStore(
    bm25_tokenization_regex=r"(?u)\b\w\w+\b",  # tokenizer สำหรับ BM25
    bm25_algorithm="BM25Okapi",                  # BM25Okapi หรือ BM25L, BM25Plus
    embedding_similarity_function="cosine",       # cosine หรือ dot_product
)

# เขียนเอกสาร
store.write_documents(documents, policy="overwrite")  # หรือ "skip", "fail"

# อ่านเอกสาร
all_docs = store.filter_documents()

# นับ
count = store.count_documents()
print(f"มี {count} เอกสาร")

# ลบ
store.delete_documents(document_ids=["id1", "id2"])
```

### 5.2 Elasticsearch Document Store

```python
from haystack_integrations.document_stores.elasticsearch import ElasticsearchDocumentStore

store = ElasticsearchDocumentStore(
    hosts="http://localhost:9200",
    index="haystack_docs",
    username="elastic",
    password="yourpassword",
    embedding_dim=768,   # ขนาด vector (ต้องตรงกับ embedding model)
)

# ติดตั้ง: pip install elasticsearch-haystack
```

### 5.3 Qdrant Document Store

```python
from qdrant_haystack import QdrantDocumentStore

store = QdrantDocumentStore(
    ":memory:",          # หรือ "http://localhost:6333" สำหรับ server
    index="documents",
    embedding_dim=768,
    recreate_index=True,
)

# ติดตั้ง: pip install qdrant-haystack
```

### 5.4 Weaviate Document Store

```python
from haystack_integrations.document_stores.weaviate import WeaviateDocumentStore

store = WeaviateDocumentStore(
    url="http://localhost:8080",
    collection_settings={
        "class": "Document",
        "vectorizer": "none",
    }
)
```

---

## 6. Embedders — แปลงข้อความเป็น Vector

ต้องใช้เมื่อต้องการ Semantic Search (ค้นหาตามความหมาย)

### 6.1 OpenAI Embedder

```python
from haystack.components.embedders import OpenAIDocumentEmbedder, OpenAITextEmbedder
import os

os.environ["OPENAI_API_KEY"] = "sk-..."

# สำหรับ embed document (ตอน indexing)
doc_embedder = OpenAIDocumentEmbedder(
    model="text-embedding-3-small",  # หรือ text-embedding-3-large
)

result = doc_embedder.run(documents=documents)
embedded_docs = result["documents"]
# แต่ละ doc จะมี doc.embedding เป็น list of float

# สำหรับ embed query (ตอนค้นหา)
text_embedder = OpenAITextEmbedder(
    model="text-embedding-3-small"
)
query_result = text_embedder.run(text="คำถามของฉัน")
query_vector = query_result["embedding"]
```

### 6.2 SentenceTransformers Embedder (ฟรี/Local)

```python
from haystack.components.embedders import (
    SentenceTransformersDocumentEmbedder,
    SentenceTransformersTextEmbedder
)

# embed documents
doc_embedder = SentenceTransformersDocumentEmbedder(
    model="sentence-transformers/all-MiniLM-L6-v2",  # 384 dim
    # หรือ "BAAI/bge-large-en-v1.5"  # 1024 dim, แม่นกว่า
    # หรือ "intfloat/multilingual-e5-large"  # รองรับหลายภาษา
    device="cpu",   # หรือ "cuda" ถ้ามี GPU
    batch_size=32,
)
doc_embedder.warm_up()  # โหลด model ลง memory

result = doc_embedder.run(documents=documents)

# embed query
text_embedder = SentenceTransformersTextEmbedder(
    model="sentence-transformers/all-MiniLM-L6-v2"
)
text_embedder.warm_up()
query_result = text_embedder.run(text="search query")
```

### เลือก Embedding Model ยังไง?

| Model | Dimension | ภาษา | ขนาด | แนะนำสำหรับ |
|-------|-----------|------|------|------------|
| `all-MiniLM-L6-v2` | 384 | EN | 22MB | Dev/Testing |
| `all-mpnet-base-v2` | 768 | EN | 420MB | Production EN |
| `multilingual-e5-large` | 1024 | 100+ ภาษา | 560MB | ภาษาไทย |
| `text-embedding-3-small` | 1536 | หลายภาษา | API | Production |
| `text-embedding-3-large` | 3072 | หลายภาษา | API | High accuracy |

---

## 7. Indexing Pipeline แบบสมบูรณ์

### 7.1 BM25 (Keyword Search) — ไม่ต้อง Embedding

```python
from haystack import Pipeline
from haystack.components.converters import PyPDFToDocument
from haystack.components.preprocessors import DocumentCleaner, DocumentSplitter
from haystack.components.writers import DocumentWriter
from haystack.document_stores.in_memory import InMemoryDocumentStore

# ตั้งค่า
store = InMemoryDocumentStore()

# สร้าง Pipeline
indexing_pipeline = Pipeline()
indexing_pipeline.add_component("converter", PyPDFToDocument())
indexing_pipeline.add_component("cleaner", DocumentCleaner())
indexing_pipeline.add_component("splitter", DocumentSplitter(
    split_by="word",
    split_length=200,
    split_overlap=20
))
indexing_pipeline.add_component("writer", DocumentWriter(document_store=store))

# เชื่อม
indexing_pipeline.connect("converter.documents", "cleaner.documents")
indexing_pipeline.connect("cleaner.documents", "splitter.documents")
indexing_pipeline.connect("splitter.documents", "writer.documents")

# รัน
indexing_pipeline.run({
    "converter": {"sources": ["document1.pdf", "document2.pdf"]}
})

print(f"Indexed {store.count_documents()} documents")
```

### 7.2 Semantic Search (Embedding-based)

```python
from haystack import Pipeline
from haystack.components.converters import TextFileToDocument
from haystack.components.preprocessors import DocumentCleaner, DocumentSplitter
from haystack.components.embedders import SentenceTransformersDocumentEmbedder
from haystack.components.writers import DocumentWriter
from haystack.document_stores.in_memory import InMemoryDocumentStore

store = InMemoryDocumentStore()

indexing_pipeline = Pipeline()
indexing_pipeline.add_component("converter", TextFileToDocument())
indexing_pipeline.add_component("cleaner", DocumentCleaner())
indexing_pipeline.add_component("splitter", DocumentSplitter(
    split_by="sentence",
    split_length=5,
    split_overlap=1
))
indexing_pipeline.add_component(
    "embedder",
    SentenceTransformersDocumentEmbedder(model="intfloat/multilingual-e5-large")
)
indexing_pipeline.add_component("writer", DocumentWriter(document_store=store))

# เชื่อม
indexing_pipeline.connect("converter.documents", "cleaner.documents")
indexing_pipeline.connect("cleaner.documents", "splitter.documents")
indexing_pipeline.connect("splitter.documents", "embedder.documents")
indexing_pipeline.connect("embedder.documents", "writer.documents")

# รัน
indexing_pipeline.run({
    "converter": {"sources": ["data.txt"]}
})
```

---

## 8. DocumentWriter

```python
from haystack.components.writers import DocumentWriter
from haystack.document_stores.in_memory import InMemoryDocumentStore

store = InMemoryDocumentStore()

writer = DocumentWriter(
    document_store=store,
    policy="overwrite"  # "overwrite", "skip", "fail"
)

# policy:
# - "overwrite": เขียนทับถ้า ID ซ้ำ
# - "skip": ข้ามถ้า ID ซ้ำ
# - "fail": error ถ้า ID ซ้ำ
```

---

## 9. Metadata Handling

```python
from haystack import Document

# ใส่ metadata ตอนสร้าง document
doc = Document(
    content="เนื้อหาบทความ",
    meta={
        "source": "company_wiki",
        "department": "engineering",
        "created_at": "2024-01-15",
        "version": "1.2",
        "language": "th",
    }
)

# Converter จะเก็บ meta จากไฟล์อัตโนมัติ
from haystack.components.converters import PyPDFToDocument

converter = PyPDFToDocument()
result = converter.run(sources=["report.pdf"])

# ดู meta ที่ auto-generate
for doc in result["documents"]:
    print(doc.meta)
    # {"file_path": "report.pdf", "page_number": 1, ...}
```

### เพิ่ม Metadata ใน Pipeline

```python
from haystack.components.routers import MetadataRouter

# route เอกสารตาม metadata
router = MetadataRouter(rules={
    "thai_docs": {"field": "meta.language", "operator": "==", "value": "th"},
    "en_docs":   {"field": "meta.language", "operator": "==", "value": "en"},
})
```

---

## 10. Save & Load Pipeline

```python
# บันทึก pipeline เป็น YAML
with open("indexing_pipeline.yaml", "w") as f:
    indexing_pipeline.dump(f)

# โหลด pipeline จาก YAML
from haystack import Pipeline

with open("indexing_pipeline.yaml", "r") as f:
    loaded_pipeline = Pipeline.load(f)
```

---

## 11. ตัวอย่างจริง: Index เอกสาร PDF บริษัท

```python
import os
from pathlib import Path
from haystack import Pipeline
from haystack.components.converters import PyPDFToDocument
from haystack.components.preprocessors import DocumentCleaner, DocumentSplitter
from haystack.components.embedders import OpenAIDocumentEmbedder
from haystack.components.writers import DocumentWriter
from haystack.document_stores.in_memory import InMemoryDocumentStore

os.environ["OPENAI_API_KEY"] = "sk-..."

# สร้าง store
store = InMemoryDocumentStore()

# สร้าง pipeline
pipeline = Pipeline()
pipeline.add_component("converter", PyPDFToDocument())
pipeline.add_component("cleaner", DocumentCleaner(
    remove_empty_lines=True,
    remove_extra_whitespaces=True,
))
pipeline.add_component("splitter", DocumentSplitter(
    split_by="passage",
    split_length=3,
    split_overlap=1,
))
pipeline.add_component("embedder", OpenAIDocumentEmbedder(
    model="text-embedding-3-small"
))
pipeline.add_component("writer", DocumentWriter(
    document_store=store,
    policy="overwrite",
))

# เชื่อม
pipeline.connect("converter.documents", "cleaner.documents")
pipeline.connect("cleaner.documents", "splitter.documents")
pipeline.connect("splitter.documents", "embedder.documents")
pipeline.connect("embedder.documents", "writer.documents")

# หาไฟล์ PDF ทั้งหมดใน folder
pdf_files = list(Path("./company_docs").glob("*.pdf"))

# รัน
pipeline.run({"converter": {"sources": pdf_files}})

print(f"✅ Indexed {store.count_documents()} document chunks")
```

---

## 12. สรุป Phase 2

```
✅ Converters แปลงไฟล์ (PDF, DOCX, HTML, TXT) → Document
✅ DocumentCleaner ทำความสะอาดข้อความ
✅ DocumentSplitter ตัดเป็น chunk (word/sentence/passage/page)
✅ split_overlap ป้องกันข้อมูลหายตรงรอยต่อ
✅ Document Stores รองรับ InMemory, Elasticsearch, Qdrant, Weaviate
✅ Embedders แปลงข้อความเป็น vector (สำหรับ semantic search)
✅ DocumentWriter เขียนลง Document Store
✅ สร้าง Indexing Pipeline โดยต่อ Component เป็น chain
```

---

**➡️ ต่อไป: Phase 3 — Retrievers & Query Pipeline**
