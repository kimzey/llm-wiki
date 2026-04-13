# 🌾 Haystack Deep Dive — Phase 2: Document Processing

> **เป้าหมาย Phase 2:** โหลดเอกสารจากหลายแหล่ง, ทำความสะอาด, และเตรียมข้อมูลก่อนเข้า DocumentStore

---

## 1. Document คืออะไร (ทบทวน)

ทุก content ใน Haystack จะถูก wrap เป็น `Document` object

```python
from haystack.dataclasses import Document

doc = Document(
    id="unique-id-123",              # Auto-generated ถ้าไม่ระบุ
    content="เนื้อหาข้อความ...",     # str | None
    blob=None,                        # ByteStream สำหรับ binary
    dataframe=None,                   # pd.DataFrame | None
    meta={                            # Metadata ใดๆ ก็ได้
        "file_path": "report.pdf",
        "page": 1,
        "author": "สมชาย"
    },
    score=None,                       # Float ใช้หลัง Retrieval
    embedding=None                    # List[float] หลัง Embedding
)
```

---

## 2. Document Converters — แปลงไฟล์เป็น Document

### 2.1 PyPDFToDocument (PDF)

```python
from haystack.components.converters import PyPDFToDocument

converter = PyPDFToDocument()

# จาก file path
result = converter.run(sources=["report.pdf", "manual.pdf"])
documents = result["documents"]

# จาก ByteStream
from haystack.dataclasses import ByteStream
stream = ByteStream(data=pdf_bytes, meta={"filename": "report.pdf"})
result = converter.run(sources=[stream])
```

### 2.2 TextFileToDocument (Plain Text)

```python
from haystack.components.converters import TextFileToDocument

converter = TextFileToDocument(
    encoding="utf-8"  # default
)
result = converter.run(sources=["readme.txt"])
```

### 2.3 HTMLToDocument (HTML/Web)

```python
from haystack.components.converters import HTMLToDocument

converter = HTMLToDocument()
result = converter.run(sources=["page.html"])

# หรือจาก URL ผ่าน LinkContentFetcher
from haystack.components.fetchers import LinkContentFetcher

fetcher = LinkContentFetcher()
streams = fetcher.run(urls=["https://example.com"])["streams"]

html_converter = HTMLToDocument()
docs = html_converter.run(sources=streams["streams"])
```

### 2.4 DOCXToDocument (Word)

```python
from haystack.components.converters import DOCXToDocument

converter = DOCXToDocument()
result = converter.run(sources=["document.docx"])
```

### 2.5 JSONConverter (JSON)

```python
from haystack.components.converters import JSONConverter

# ระบุ field ที่จะดึงมาเป็น content
converter = JSONConverter(
    content_key="text",      # field หลักเป็น content
    meta_fields_to_embed=["title", "category"]  # field ที่ embed รวมด้วย
)
result = converter.run(sources=["data.json"])
```

### 2.6 AzureOCRDocumentConverter (Scanned PDF/Image)

```python
# ต้องติดตั้ง: pip install azure-ai-formrecognizer
from haystack.components.converters import AzureOCRDocumentConverter

converter = AzureOCRDocumentConverter(
    azure_endpoint="https://xxx.cognitiveservices.azure.com/",
    azure_key="your-key"
)
result = converter.run(sources=["scanned_doc.pdf"])
```

### สรุป Converter ที่ใช้บ่อย

| Converter | Input | Package |
|---|---|---|
| `PyPDFToDocument` | `.pdf` | `pypdf` |
| `PDFMinerToDocument` | `.pdf` (layout-aware) | `pdfminer.six` |
| `AzureOCRDocumentConverter` | Scanned PDF, Image | `azure-ai-formrecognizer` |
| `HTMLToDocument` | `.html` | built-in |
| `TextFileToDocument` | `.txt`, `.md` | built-in |
| `DOCXToDocument` | `.docx` | `python-docx` |
| `JSONConverter` | `.json` | built-in |
| `CSVToDocument` | `.csv` | built-in |
| `MarkdownToDocument` | `.md` | built-in |

---

## 3. Document Preprocessors — ทำความสะอาดและตัดแบ่ง

### 3.1 DocumentCleaner (ทำความสะอาด)

```python
from haystack.components.preprocessors import DocumentCleaner

cleaner = DocumentCleaner(
    remove_empty_lines=True,           # ลบบรรทัดว่าง
    remove_extra_whitespaces=True,     # ลบ whitespace ซ้ำ
    remove_repeated_substrings=False,  # ลบ text ซ้ำ (เช่น header/footer)
    keep_id=False,                     # เก็บ ID เดิมหรือสร้างใหม่
    unicode_normalization="NFC"        # Normalize unicode
)

result = cleaner.run(documents=documents)
clean_docs = result["documents"]
```

### 3.2 DocumentSplitter (ตัดแบ่งเอกสาร) ⭐ สำคัญมาก

การตัดเอกสารเป็น chunk เล็กๆ เป็นหัวใจสำคัญของ RAG

```python
from haystack.components.preprocessors import DocumentSplitter

splitter = DocumentSplitter(
    split_by="word",        # "word" | "sentence" | "passage" | "page" | "function"
    split_length=200,       # จำนวน unit ต่อ chunk
    split_overlap=20,       # จำนวน unit ที่ overlap กัน
    split_threshold=0,      # chunk ที่เล็กกว่านี้จะ merge กับ chunk ก่อน
)

result = splitter.run(documents=clean_docs)
chunks = result["documents"]

# ตรวจสอบผลลัพธ์
for i, chunk in enumerate(chunks[:3]):
    print(f"Chunk {i}: {len(chunk.content.split())} words")
    print(f"  Meta: {chunk.meta}")
    print()
```

### เลือก split_by อย่างไร?

| split_by | เหมาะกับ | ข้อดี | ข้อเสีย |
|---|---|---|---|
| `word` | เอกสารทั่วไป | ควบคุม size ง่าย | ตัดกลางประโยคได้ |
| `sentence` | ข้อความบทความ | อ่านได้ดี | size ไม่แน่นอน |
| `passage` | Long-form content | บริบทสมบูรณ์ | chunk ใหญ่เกิน |
| `page` | PDF | ตาม structure | page ยาวมาก |
| `function` | Custom logic | Flexible | ต้องเขียนเอง |

### 3.3 การตัดแบบ Custom Function

```python
def custom_split(text: str) -> list[str]:
    """ตัดตาม header ของเอกสาร"""
    import re
    sections = re.split(r'\n#{1,3}\s', text)
    return [s.strip() for s in sections if s.strip()]

splitter = DocumentSplitter(
    split_by="function",
    splitting_function=custom_split
)
```

---

## 4. Indexing Pipeline — ไปป์ไลน์บันทึกเอกสาร

นี่คือ Pipeline สำหรับนำเอกสารเข้า DocumentStore

### 4.1 Simple Indexing Pipeline (BM25)

```python
from haystack import Pipeline
from haystack.components.converters import PyPDFToDocument
from haystack.components.preprocessors import DocumentCleaner, DocumentSplitter
from haystack.components.writers import DocumentWriter
from haystack.document_stores.in_memory import InMemoryDocumentStore

# DocumentStore
document_store = InMemoryDocumentStore()

# สร้าง Pipeline
indexing_pipeline = Pipeline()
indexing_pipeline.add_component("converter",  PyPDFToDocument())
indexing_pipeline.add_component("cleaner",    DocumentCleaner())
indexing_pipeline.add_component("splitter",   DocumentSplitter(
    split_by="word",
    split_length=200,
    split_overlap=20
))
indexing_pipeline.add_component("writer", DocumentWriter(document_store=document_store))

# เชื่อมต่อ
indexing_pipeline.connect("converter.documents", "cleaner.documents")
indexing_pipeline.connect("cleaner.documents",   "splitter.documents")
indexing_pipeline.connect("splitter.documents",  "writer.documents")

# รัน
result = indexing_pipeline.run({
    "converter": {
        "sources": ["doc1.pdf", "doc2.pdf"],
        "meta": {"batch": "v1"}  # meta ที่ inject เข้าทุก Document
    }
})

print(f"บันทึก {result['writer']['documents_written']} documents")
```

### 4.2 Indexing Pipeline with Embedding (Vector Search)

```python
from haystack.components.embedders import SentenceTransformersDocumentEmbedder
from haystack.document_stores.in_memory import InMemoryDocumentStore

document_store = InMemoryDocumentStore()

indexing_pipeline = Pipeline()
indexing_pipeline.add_component("converter", PyPDFToDocument())
indexing_pipeline.add_component("cleaner",   DocumentCleaner())
indexing_pipeline.add_component("splitter",  DocumentSplitter(
    split_by="word", split_length=200, split_overlap=20
))
indexing_pipeline.add_component("embedder",  SentenceTransformersDocumentEmbedder(
    model="BAAI/bge-m3"  # รองรับหลายภาษารวมถึงภาษาไทย
))
indexing_pipeline.add_component("writer",    DocumentWriter(
    document_store=document_store,
    policy="overwrite"  # "none" | "overwrite" | "skip"
))

# เชื่อมต่อ
indexing_pipeline.connect("converter.documents", "cleaner.documents")
indexing_pipeline.connect("cleaner.documents",   "splitter.documents")
indexing_pipeline.connect("splitter.documents",  "embedder.documents")
indexing_pipeline.connect("embedder.documents",  "writer.documents")

indexing_pipeline.run({"converter": {"sources": ["report.pdf"]}})
```

---

## 5. Metadata Enrichment — เพิ่ม Metadata ให้ Document

Metadata ช่วยใน **filtering** ตอน retrieval

```python
from haystack.components.converters import PyPDFToDocument

converter = PyPDFToDocument()

# เพิ่ม meta ตอน run
result = converter.run(
    sources=["q1_report.pdf", "q2_report.pdf"],
    meta={
        "company": "ACME Corp",
        "year": 2024,
        "document_type": "financial_report"
    }
)

# แต่ละ Document จะมี meta นี้รวมกับ auto-generated meta
for doc in result["documents"]:
    print(doc.meta)
    # {'company': 'ACME Corp', 'year': 2024, 'document_type': 'financial_report',
    #  'file_path': 'q1_report.pdf', 'page_number': 1, ...}
```

---

## 6. ตัวอย่าง Real-world: Index เอกสารจากหลายแหล่ง

```python
import os
from pathlib import Path
from haystack import Pipeline
from haystack.components.converters import (
    PyPDFToDocument, TextFileToDocument, HTMLToDocument
)
from haystack.components.routers import FileTypeRouter
from haystack.components.joiners import DocumentJoiner
from haystack.components.preprocessors import DocumentCleaner, DocumentSplitter
from haystack.components.embedders import SentenceTransformersDocumentEmbedder
from haystack.components.writers import DocumentWriter
from haystack.document_stores.in_memory import InMemoryDocumentStore

document_store = InMemoryDocumentStore()

# FileTypeRouter จะ route ไฟล์ตาม MIME type
pipeline = Pipeline()
pipeline.add_component("router", FileTypeRouter(mime_types=[
    "text/plain",
    "application/pdf",
    "text/html"
]))
pipeline.add_component("pdf_converter",  PyPDFToDocument())
pipeline.add_component("txt_converter",  TextFileToDocument())
pipeline.add_component("html_converter", HTMLToDocument())
pipeline.add_component("joiner",   DocumentJoiner())      # รวม documents กลับมา
pipeline.add_component("cleaner",  DocumentCleaner())
pipeline.add_component("splitter", DocumentSplitter(split_by="word", split_length=150))
pipeline.add_component("embedder", SentenceTransformersDocumentEmbedder(model="BAAI/bge-m3"))
pipeline.add_component("writer",   DocumentWriter(document_store=document_store))

# เชื่อมต่อ
pipeline.connect("router.text/plain",         "txt_converter.sources")
pipeline.connect("router.application/pdf",    "pdf_converter.sources")
pipeline.connect("router.text/html",          "html_converter.sources")
pipeline.connect("pdf_converter.documents",   "joiner.documents")
pipeline.connect("txt_converter.documents",   "joiner.documents")
pipeline.connect("html_converter.documents",  "joiner.documents")
pipeline.connect("joiner.documents",          "cleaner.documents")
pipeline.connect("cleaner.documents",         "splitter.documents")
pipeline.connect("splitter.documents",        "embedder.documents")
pipeline.connect("embedder.documents",        "writer.documents")

# โหลดไฟล์ทุกประเภทในโฟลเดอร์
all_files = list(Path("./data").glob("**/*"))
files = [str(f) for f in all_files if f.is_file()]

result = pipeline.run({"router": {"sources": files}})
print(f"✅ Index แล้ว: {result['writer']['documents_written']} chunks")
```

---

## 7. Best Practices สำหรับ Document Processing

### Chunk Size Guidelines

```
เอกสารทั่วไป:     split_length=200-300 words, overlap=20-30
FAQ/Q&A:          split_length=100-150 words (คำถาม-คำตอบ)
Legal/Technical:  split_length=300-500 words (context ต้องสมบูรณ์)
Code/Technical:   split_by="passage" (ตาม function/class)
```

### การเลือก Embedding Model สำหรับภาษาไทย

```python
# แนะนำสำหรับภาษาไทยและ multilingual
models = {
    "BAAI/bge-m3":             "ดีที่สุดสำหรับ multilingual รวมไทย",
    "intfloat/multilingual-e5-large": "แม่นยำสูง ใช้ resource เยอะ",
    "sentence-transformers/paraphrase-multilingual-mpnet-base-v2": "เบา เร็ว"
}
```

---

## 8. สรุป Phase 2

| หัวข้อ | สิ่งที่ได้เรียนรู้ |
|---|---|
| Document Converters | แปลง PDF, HTML, DOCX, TXT เป็น Document |
| DocumentCleaner | ทำความสะอาด whitespace, unicode |
| DocumentSplitter | ตัดเป็น chunk ด้วย word/sentence/page/function |
| Indexing Pipeline | เชื่อม Converter → Cleaner → Splitter → Writer |
| Metadata | เพิ่ม meta เพื่อ filter ตอน retrieval |
| Multi-source | FileTypeRouter + DocumentJoiner สำหรับหลาย format |

---

## 📚 Phase ถัดไป

- **Phase 3:** Embedding & Vector Search — สร้าง Semantic Search
