# OpenRAG Ingestion Flow — อธิบาย Component ทั้งหมด

## สรุปภาพรวม

OpenRAG มี **2 เส้นทาง** สำหรับ ingestion:

| เส้นทาง | ตัวแปร Env | ใครทำ Chunking |
|---|---|---|
| **Langflow** (default) | `DISABLE_INGEST_WITH_LANGFLOW=false` | Langflow **Split Text** component |
| **Backend Python** | `DISABLE_INGEST_WITH_LANGFLOW=true` | Python code ใน `processors.py` + `document_processing.py` |

---

## เส้นทาง 1: Langflow Ingestion (Default)

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  DATA SOURCE                                                                │
│  (Local File / Cloud Connector / S3)                                       │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ file path / bytes
                               ▼
┌──────────────────────────────────────────┐
│  [1] Docling Serve                       │
│  component: DoclingRemote                │
│  connects to: http://localhost:5001      │
│  output: DoclingDocument (structured)   │
│  → ทำ OCR, table parsing, image parsing │
└──────────────────────────────┬───────────┘
                               │ DataFrame
                               ▼
┌──────────────────────────────────────────┐
│  [2] Export DoclingDocument              │
│  component: ExportDoclingDocument        │
│  → แปลง DoclingDocument → Markdown text │
│  → image placeholder แทนรูปภาพ          │
│  output: DataFrame (column: text)        │
└──────────────────────────────┬───────────┘
                               │ DataFrame
                               ▼
┌──────────────────────────────────────────┐
│  [3a] DataFrame Operations #1            │
│  → เพิ่ม column: `filename`              │
└──────────────────────────────┬───────────┘
                               ▼
┌──────────────────────────────────────────┐
│  [3b] DataFrame Operations #2            │
│  → เพิ่ม column: `file_size`             │
└──────────────────────────────┬───────────┘
                               ▼
┌──────────────────────────────────────────┐
│  [3c] DataFrame Operations #3            │
│  → เพิ่ม column: `mimetype`              │
└──────────────────────────────┬───────────┘
                               │ DataFrame (text + filename + file_size + mimetype)
                               ▼
┌─────────────────────────────────────────────────────┐
│  [4] Split Text  ← ★ CHUNKING อยู่ที่นี่ ★           │
│  component: SplitText                               │
│  settings (ปรับได้ใน Settings UI):                  │
│    - Chunk size:    1000 characters (default)       │
│    - Chunk overlap: 200 characters (default)        │
│  output: DataFrame (แต่ละ row = 1 chunk)            │
└───────────────┬─────────────────────────────────────┘
                │                    │
                │ ingest_data         │ (chunks ถูกส่งไป embed และ index)
                ▼                    ▼
┌───────────────────────────────────────────────────────────────────────────┐
│  [6] OpenSearch (Multi-Model Multi-Embedding)                             │
│  component: OpenSearchVectorStoreComponentMultimodalMultiEmbedding        │
│  - รับ ingest_data (chunks) จาก Split Text                                │
│  - รับ embeddings จาก Embedding Models (ได้สูงสุด 3 models พร้อมกัน)     │
│  - รับ Document Metadata จาก AdvancedDynamicFormBuilder                   │
│  - เก็บ chunk + vector + metadata ลง OpenSearch index                     │
│  auth: JWT (default) หรือ basic auth                                      │
│  env: OPENSEARCH_URL, OPENSEARCH_INDEX_NAME, JWT                          │
└───────────────────────────────────────────────────────────────────────────┘
        ▲               ▲               ▲               ▲
        │               │               │               │
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────────────────────────────┐
│ [5a]         │ │ [5b]         │ │ [5c]         │ │ [Metadata Path]                      │
│ Embedding    │ │ Embedding    │ │ Embedding    │ │                                      │
│ Model #1     │ │ Model #2     │ │ Model #3     │ │ TextInput (document_id)              │
│ (primary)    │ │ (optional)   │ │ (optional)   │ │ TextInput (owner)                    │
│              │ │              │ │              │ │ TextInput (owner_email)              │
│ default:     │ │              │ │              │ │ TextInput (owner_name)               │
│ text-embed   │ │              │ │              │ │ TextInput (source_url)               │
│ ding-3-small │ │              │ │              │ │   ↓                                  │
│              │ │              │ │              │ │ AdvancedDynamicFormBuilder            │
│              │ │              │ │              │ │ → รวม inputs → Data object           │
│              │ │              │ │              │ │ → docs_metadata → OpenSearch         │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────────────────────────────┘
```

---

## เส้นทาง 2: Backend Python Ingestion (`DISABLE_INGEST_WITH_LANGFLOW=true`)

### Flow Diagram

```
┌──────────────────────────────────────────┐
│  DATA SOURCE                             │
│  file_path (local / connector / S3)      │
└──────────────────────────────┬───────────┘
                               │
                               ▼
┌──────────────────────────────────────────┐
│  processors.py                           │
│  class: DocumentFileProcessor           │
│         ConnectorFileProcessor          │
│         S3FileProcessor                  │
│  → hash_id(file_path) เช็ค duplicate     │
│  → check_document_exists() in OpenSearch │
│  → ถ้ามีแล้ว: return {status: unchanged} │
└──────────────────────────────┬───────────┘
                               │
            ┌──────────────────┴──────────────┐
            │ .txt / .md file?                │
            ▼                                 ▼
    ┌───────────────────┐          ┌────────────────────────┐
    │ process_text_file │          │ docling_client         │
    │ (simple, fast)    │          │ convert_file()         │
    │                   │          │ → POST to Docling Serve│
    │ ★ CHUNKING ★       │          │   API (:5001)          │
    │ Split by \n\n     │          │ → full DoclingDocument │
    │ ~1000 chars/chunk │          └──────────┬─────────────┘
    │ no overlap        │                     │
    └────────┬──────────┘                     ▼
             │                    ┌────────────────────────┐
             │                    │ extract_relevant()     │
             │                    │ document_processing.py │
             │                    │                        │
             │                    │ ★ CHUNKING ★            │
             │                    │ Group texts by page_no │
             │                    │ → 1 chunk = 1 page     │
             │                    │ Tables → tab-separated │
             │                    │   text, 1 chunk/table  │
             └──────────┬─────────┘
                        │ slim_doc["chunks"] = [{page, type, text}, ...]
                        ▼
┌──────────────────────────────────────────────────────────┐
│  chunk_texts_for_embeddings()                            │
│  services/document_service.py                            │
│  → batch chunks ไม่เกิน 8000 tokens ต่อ batch            │
└──────────────────────────────┬───────────────────────────┘
                               │ batches
                               ▼
┌──────────────────────────────────────────────────────────┐
│  clients.patched_embedding_client.embeddings.create()    │
│  → เรียก Embedding Model API (OpenAI / Ollama / etc.)    │
│  → ได้ vector ต่อ chunk                                   │
└──────────────────────────────┬───────────────────────────┘
                               │ (chunk, vector) pairs
                               ▼
┌──────────────────────────────────────────────────────────┐
│  opensearch_client.index()                               │
│  → index แต่ละ chunk เป็น doc แยกกัน                     │
│  → chunk_id = "{file_hash}_{i}"                          │
│  fields ที่เก็บ:                                          │
│    document_id, filename, mimetype, page, text           │
│    embedding_<model_name> (vector field)                 │
│    embedding_model, embedding_dimensions                 │
│    file_size, connector_type, indexed_time               │
│    owner, allowed_users, allowed_groups                  │
│    owner_name, owner_email                               │
└──────────────────────────────────────────────────────────┘
```

---

## ตอบคำถาม: Chunking อยู่ที่ไหน?

### Langflow Path
**Chunking = Split Text component** ใน Langflow flow
- ไฟล์: `flows/ingestion_flow.json` (node ID: `SplitText-QIKhg`)
- ปรับได้ที่ UI: **Settings → Knowledge Ingest → Chunk size / Chunk overlap**
- Default: **1000 characters**, overlap **200 characters**

### Backend Path (non-Langflow)
**Chunking เกิดใน Python code โดยตรง** ใน 2 ที่:

| ไฟล์ | Function | กลยุทธ์ |
|---|---|---|
| `src/utils/document_processing.py` | `extract_relevant()` | 1 chunk = 1 page (จาก Docling) + tables แยก chunk |
| `src/utils/document_processing.py` | `process_text_file()` | split by paragraph (`\n\n`), ~1000 chars/chunk, **ไม่มี overlap** |

> **หมายเหตุ**: Backend path ไม่ใช้ Split Text component และไม่มี overlap ใน `process_text_file()`
> ถ้าต้องการควบคุม chunk size/overlap อย่างละเอียด ให้ใช้ Langflow path (default)

---

## Component Summary Table

| # | Component | ชื่อใน Langflow | หน้าที่ | Output |
|---|---|---|---|---|
| 1 | **Docling Serve** | `DoclingRemote` | แปลง file → structured document (OCR, tables, images) | `DataFrame` |
| 2 | **Export DoclingDocument** | `ExportDoclingDocument` | แปลง DoclingDocument → Markdown | `DataFrame` |
| 3a | **DataFrame Operations #1** | `DataFrameOperations-1BWXB` | เพิ่ม column `filename` | `DataFrame` |
| 3b | **DataFrame Operations #2** | `DataFrameOperations-9vMrp` | เพิ่ม column `file_size` | `DataFrame` |
| 3c | **DataFrame Operations #3** | `DataFrameOperations-N80fC` | เพิ่ม column `mimetype` | `DataFrame` |
| 4 | **Split Text** ★ | `SplitText-QIKhg` | **Chunk text** ตาม size/overlap | `DataFrame` (rows = chunks) |
| 5a-c | **Embedding Model x3** | `EmbeddingModel-*` | สร้าง vector embeddings | `Embeddings` |
| 6 | **OpenSearch (Multi-Model)** | `OpenSearchVectorStoreComponentMultimodalMultiEmbedding-By9U4` | เก็บ chunks + vectors ลง index | — |
| M1 | **Text Input** (x6) | `TextInput-*` | รับ metadata: `document_id`, `owner`, `owner_email`, `owner_name`, `source_url`, `allowed_users/groups` | `Message` |
| M2 | **AdvancedDynamicFormBuilder** | `AdvancedDynamicFormBuilder-81Exw` | รวม Text Inputs → structured `Data` object | `Data` |

---

## ข้อมูลที่ถูกเก็บใน OpenSearch ต่อ Chunk

```json
{
  "document_id": "sha256-hash-of-file",
  "filename": "example.pdf",
  "mimetype": "application/pdf",
  "page": 3,
  "text": "...chunk content...",
  "embedding_text-embedding-3-small": [0.123, -0.456, ...],
  "embedding_model": "text-embedding-3-small",
  "embedding_dimensions": 1536,
  "file_size": 204800,
  "connector_type": "local",
  "indexed_time": "2026-03-18T10:00:00",
  "owner": "user-id",
  "owner_name": "Kim Zey",
  "owner_email": "kim@example.com",
  "allowed_users": [],
  "allowed_groups": []
}
```

---

## Source Files ที่เกี่ยวข้อง

| ไฟล์ | หน้าที่ |
|---|---|
| `flows/ingestion_flow.json` | Langflow flow definition (nodes + edges) |
| `src/models/processors.py` | TaskProcessor classes, orchestration logic |
| `src/utils/document_processing.py` | `extract_relevant()`, `process_text_file()` — chunking สำหรับ backend path |
| `src/utils/docling_client.py` | Docling Serve API client |
| `src/services/document_service.py` | `chunk_texts_for_embeddings()` — batch splitter for embedding API |
| `docs/docs/_partial-ingestion-flow.mdx` | Official docs description |
| `docs/docs/core-components/ingestion-configure.mdx` | Chunk size/overlap settings docs |
