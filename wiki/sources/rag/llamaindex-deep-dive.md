---
title: "LlamaIndex & RAG Deep Dive — Sellsuki Knowledge Bot"
type: source
source_file: raw/notes/tool-rag/llamaindex-deep-dive.md
url: ""
published: 2026-01-01
tags: [llamaindex, rag, llamaparse, llamahub, llamatrace, paradedb, dragonfly, sellsuki]
related: [wiki/concepts/llamaindex-framework.md, wiki/concepts/rag-chunking-strategies.md, wiki/concepts/hybrid-search-bm25-vector.md, wiki/concepts/semantic-caching.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/rag/llamaindex-deep-dive.md|Original file]]

## สรุป

Deep Dive LlamaIndex ครบทุกส่วน: Ecosystem (LlamaHub, LlamaParse, LlamaTrace), RAG Pipeline, Architecture สำหรับ Sellsuki Knowledge Bot, Chunking Strategies ทั้ง 4 แบบ, Metadata & Filtering, ParadeDB Schema, Dragonfly Cache, Smart Update Pipeline

## ประเด็นสำคัญ

### LlamaIndex Ecosystem

| Component | คือ | ใช้เมื่อ |
|-----------|-----|---------|
| **llama-index-core** | หัวใจ: nodes, indexes, engines | ทุก use case |
| **LlamaHub** | 300+ community connectors | ดึงข้อมูลจาก Notion, Confluence, Google Drive |
| **LlamaParse** | Parse เอกสารซับซ้อน (PDF ที่มีตาราง, รูป) | PDF ยาก, ดีกว่า PyPDF2/pdfplumber มาก |
| **LlamaTrace / Arize Phoenix** | Observability — retrieval quality, latency | Production monitoring |
| **LlamaAgents / Workflows** | Multi-step AI workflows, multi-agent | Router Agent เลือก index ที่ถูกต้อง |

**LlamaParse** — รองรับภาษาไทย:
```python
parser = LlamaParse(result_type="markdown", language="th", api_key="llx-...")
docs = parser.load_data("hr_policy.pdf")
# Preserve ตาราง + headers ได้ดีมาก
```

### Chunking Strategies ทั้ง 4 แบบ (สำหรับ Sellsuki)

| Strategy | Code | เหมาะกับ Sellsuki docs |
|---------|------|----------------------|
| Fixed-Size | `SentenceSplitter(chunk_size=512, overlap=50)` | Dev Standards (code snippets) |
| Semantic ⭐ | `SemanticSplitterNodeParser(breakpoint_percentile_threshold=95)` | Product Docs, HR Policy |
| Hierarchical ⭐⭐ | `HierarchicalNodeParser.from_defaults(chunk_sizes=[2048, 512, 128])` + `AutoMergingRetriever` | HR Policy, Onboarding |
| Sentence Window | `SentenceWindowNodeParser(window_size=3)` + `MetadataReplacementPostProcessor` | IT Guide (ขั้นตอน) |

**Hierarchical + AutoMergingRetriever** — ดึง leaf nodes หลายอัน → auto merge เป็น parent → ได้ context ครบโดยไม่ส่ง tokens เยอะ

### Metadata Schema สำหรับ Sellsuki

```python
metadata={
    "doc_id": "hr-policy-2024-v3",
    "section": "Leave Policy",
    "department": ["all"],          # array
    "access_level": "public",       # public | internal | confidential
    "category": "hr",               # hr | it | product | dev | general
    "tags": ["leave", "annual", "วันลา"],
    "language": "th",
    "content_hash": "sha256:...",   # ใช้เช็คว่าเนื้อหาเปลี่ยนไหม
    "is_latest": True,
}
```

**Auto Metadata Extraction ด้วย LLM:**
```python
pipeline = IngestionPipeline(transformations=[
    SentenceWindowNodeParser.from_defaults(window_size=3),
    TitleExtractor(nodes=5),
    QuestionsAnsweredExtractor(questions=3, llm=llm),  # generate Q&A อัตโนมัติ
    KeywordExtractor(keywords=5),
    embed_model,
])
```

### Sellsuki Knowledge Bot Architecture

```
[Interfaces: LINE/Slack/Web/Cursor]
          ↓
[FastAPI API Gateway]
          ↓
[LlamaIndex Query Engine]
  → Semantic Cache (Dragonfly) → HIT
  → Query Router: HR/IT/Product/Dev/General
  → Hybrid Retrieve (ParadeDB: Vector + BM25)
  → Reranker (Cohere)
  → LLM (Claude / GPT-4o) → Answer + Source Nodes
          ↓
[Response: {answer, sources, confidence}]
```

### ParadeDB Schema Design

```sql
CREATE TABLE document_chunks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    doc_id TEXT NOT NULL,
    content TEXT NOT NULL,
    content_hash TEXT NOT NULL,
    embedding vector(1536),
    category TEXT,
    department TEXT[],
    access_level TEXT DEFAULT 'public',
    is_latest BOOLEAN DEFAULT true,
    ...
);
-- BM25 index (ICU tokenizer สำหรับภาษาไทย)
CREATE INDEX USING bm25(id, content, tags, section)
WITH (text_fields='{"content": {"tokenizer": {"type": "icu"}}}');
-- HNSW index (เร็วกว่า IVFFlat)
CREATE INDEX USING hnsw (embedding vector_cosine_ops) WITH (m=16, ef_construction=64);
```

### RAG vs Fine-tuning

| | Fine-tuning | RAG |
|--|-------------|-----|
| อัปเดตข้อมูล | ต้อง train ใหม่ | แค่เพิ่ม/แก้ document |
| ค่าใช้จ่าย | สูงมาก (GPU) | ถูกกว่ามาก |
| อ้างอิงแหล่งที่มา | ทำยาก | ทำได้ง่าย |
| ข้อมูล confidential | เสี่ยง data leak | ข้อมูลอยู่ใน DB |

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- LlamaIndex `RouterQueryEngine` + `RouterRetriever`: เลือก index ที่เหมาะสมตาม query — เหมาะถ้ามีหลาย departments
- `AutoMergingRetriever`: ถ้าดึง leaf nodes หลายอัน → merge เป็น parent node อัตโนมัติ
- Hybrid search RRF (Reciprocal Rank Fusion): รวม vector + BM25 score โดยไม่ต้องกำหนด weight ล่วงหน้า
- `MetadataFilter(key="department", value="engineering", operator=FilterOperator.CONTAINS)` — filter array field ได้

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/llamaindex-framework|LlamaIndex Framework]]
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]]
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search — BM25 + Vector]]
- [[wiki/concepts/semantic-caching|Semantic Caching]]
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG — Retrieval-Augmented Generation]]
