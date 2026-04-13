---
title: "OpenRAG — Ingestion Paths & Chunking Analysis"
type: source
source_file: raw/notes/openrag/docs-lean/ingestion-flow-explained.md
published: 2026-03-18
tags: [openrag, ingestion, langflow, chunking, character-text-splitter, markdown]
related: [wiki/concepts/openrag-platform.md, wiki/concepts/rag-chunking-strategies.md, wiki/concepts/langflow-visual-workflow.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/openrag/docs-lean/ingestion-flow-explained.md|Original file]]
> **Additional**: [[../../raw/notes/openrag/docs-lean/ingestion-two-paths-explained.md|Two Paths Explained]]
> **Additional**: [[../../raw/notes/openrag/docs-lean/md-file-and-splitter-analysis.md|MD File & Splitter Analysis]]

## สรุป

วิเคราะห์ 2 ingestion paths ใน OpenRAG (Langflow vs Backend Python) และ CharacterTextSplitter behavior โดยละเอียด

## ประเด็นสำคัญ

### 2 Ingestion Paths

```
DISABLE_INGEST_WITH_LANGFLOW=false  → Langflow pipeline  (default ✅)
DISABLE_INGEST_WITH_LANGFLOW=true   → Backend Python pipeline
```

ไม่ได้ทำงานพร้อมกัน — เป็น switch เลือก path ต่อ request

### Path 1: Langflow (Default)

```
File
  → [1] DoclingRemote (http://localhost:5001) — parse OCR/table
  → [2] ExportDoclingDocument — DoclingDocument → Markdown text
  → [3a-c] DataFrameOperations (เพิ่ม filename, file_size, mimetype)
  → [4] SplitText-QIKhg ★ CHUNKING ★ (CharacterTextSplitter)
      chunk_size=1000, overlap=200, separator="\n"
  → [5a-c] EmbeddingModel (ได้สูงสุด 3 models พร้อมกัน)
  → [6] OpenSearchVectorStore (index + ACL metadata)
```

Metadata path: 8 TextInputs → AdvancedDynamicFormBuilder → OpenSearch
- OWNER, OWNER_NAME, OWNER_EMAIL, ALLOWED_USERS, ALLOWED_GROUPS
- CONNECTOR_TYPE, DOCUMENT_ID, SOURCE_URL

### Path 2: Backend Python

| ไฟล์ประเภท | ผ่าน Docling? | Chunking Strategy | Overlap |
|-----------|--------------|------------------|---------|
| PDF/DOCX/PPT etc. | ✅ | 1 page = 1 chunk + tables แยก | ❌ ไม่มี |
| .txt / .md | ❌ bypass | split `\n\n`, ~1000 chars | ❌ ไม่มี |

Code: `src/utils/document_processing.py`
- `extract_relevant()` — PDF/DOCX (page-based chunks)
- `process_text_file()` — .txt/.md (paragraph-based, hardcoded)
- `chunk_texts_for_embeddings()` — safety net ≤8000 tokens/batch (ไม่ใช่ chunking จริง)

### CharacterTextSplitter วิเคราะห์

OpenRAG ใช้ `CharacterTextSplitter` (ไม่ใช่ `RecursiveCharacterTextSplitter`)

```python
# Langflow SplitText node (SplitText-QIKhg)
CharacterTextSplitter(
    chunk_size=1000,    # characters
    chunk_overlap=200,
    separator="\n",     # single separator เท่านั้น
)
```

**Algorithm:**
1. Split by separator (`"\n"`)
2. Merge splits จนถึง chunk_size (1000)
3. Overlap 200 chars จาก chunk ก่อนหน้า

**ปัญหาที่พบ:**

| กรณี | ผลกระทบ |
|------|---------|
| Table ถูกตัดกลาง | Header row กับ data rows อาจอยู่คนละ chunk |
| Code block ถูกตัด | ` ```python ` กับ ` ``` ` อาจแยก chunk |
| Single split > chunk_size | เก็บทั้งก้อนเลย (ไม่ตัดต่อ) |
| Markdown structure | ไม่รู้จัก header/list/table — ตัดทุก `\n` |

**ทางแก้:** เปลี่ยน separator จาก `\n` → `\n\n` ใน Langflow UI หรือใช้ MarkdownTextSplitter

### เปรียบเทียบ 3 splitters

| Splitter | OpenRAG ใช้? | Logic |
|----------|-------------|-------|
| CharacterTextSplitter | ✅ Langflow | 1 separator เท่านั้น |
| RecursiveCharacterTextSplitter | ❌ | ลอง separators ตามลำดับ ["\n\n", "\n", " ", ""] |
| MarkdownTextSplitter | ❌ | รู้จัก Markdown structure ("#", "##", "###") |

LangChain เองแนะนำ RecursiveCharacterTextSplitter เป็น default

### ไฟล์ .md กับ Docling

| Path | .md ผ่าน Docling? | Chunking |
|------|-----------------|---------|
| Langflow (default) | ✅ ผ่านทุกไฟล์ | CharacterTextSplitter `"\n"`, overlap=200 |
| Backend | ❌ bypass | `process_text_file()`, `"\n\n"`, ไม่มี overlap |

### ข้อมูล Chunk ที่เก็บใน OpenSearch

```json
{
  "document_id": "sha256-hash",
  "filename": "example.pdf",
  "page": 3,
  "text": "...chunk content...",
  "embedding_text-embedding-3-small": [0.123, -0.456, ...],
  "embedding_model": "text-embedding-3-small",
  "embedding_dimensions": 1536,
  "owner": "user-id",
  "allowed_users": [],
  "allowed_groups": []
}
```

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/openrag-platform|OpenRAG Platform]] — platform overview
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]] — chunking options comparison
- [[wiki/concepts/langflow-visual-workflow|Langflow]] — Langflow flow components
