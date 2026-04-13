---
title: "RAG Chunking Strategies"
type: concept
tags: [rag, chunking, llamaindex, langchain, semantic-split, hierarchical-split]
sources: [rag-decision-guide.md, llamaindex-deep-dive.md, llamaindex-full-guide.md, langchain-llamaindex-deep-dive.md, langchain-rag-guide.md]
related: [wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/llamaindex-framework.md, wiki/concepts/langchain-framework.md]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

Chunking คือการตัดเอกสารใหญ่เป็นชิ้นเล็ก ๆ ก่อน embedding และ retrieval — วิธีที่เลือกส่งผลโดยตรงต่อคุณภาพของ RAG: Recall (หาได้ครบไหม), Precision (หาถูกต้องไหม), และ Relevance (ได้ chunk ที่เกี่ยวข้องจริงไหม)

## อธิบาย

เอกสารขนาดใหญ่ (100+ pages) ไม่สามารถ embed ไปทีเดียวได้ ต้องแบ่งเป็น chunks ก่อน แต่ถ้าแบ่งผิดวิธี:
- ตัดกลางประโยค → context ขาด
- ตัดกลาง concept → LLM ไม่เข้าใจ
- chunk ใหญ่เกินไป → retrieval ไม่แม่น (noise เยอะ)
- chunk เล็กเกินไป → ขาด context

ทั้ง LangChain และ LlamaIndex มี TextSplitters / NodeParsers หลายแบบให้เลือก

## ประเด็นสำคัญ

### 1. Fixed-Size Chunking (Baseline)

แบ่งตามจำนวน characters/tokens ที่กำหนด — ง่ายที่สุด, เร็ว, แต่ตัด context ได้

**LangChain:**
```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,      # characters
    chunk_overlap=50,    # overlap เพื่อไม่ให้ context ขาด
    separators=["\n\n", "\n", ".", " "]
)
```

**LlamaIndex:**
```python
from llama_index.core.node_parser import SentenceSplitter

splitter = SentenceSplitter(
    chunk_size=512,       # tokens
    chunk_overlap=50,
)
```

เหมาะกับ: เอกสารทั่วไป, เริ่มต้น
ข้อเสีย: ตัดกลางประโยคได้, ไม่เข้าใจ structure

---

### 2. Semantic Chunking (แนะนำสำหรับเริ่ม)

แทนที่จะตัดตามขนาด → ตัดเมื่อ **ความหมายเปลี่ยน**

**LangChain:**
```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

splitter = SemanticChunker(
    OpenAIEmbeddings(),
    breakpoint_threshold_type="percentile",  # หรือ "standard_deviation"
    breakpoint_threshold_amount=95,         # percentile
)
```

**LlamaIndex:**
```python
from llama_index.core.node_parser import SemanticSplitterNodeParser

splitter = SemanticSplitterNodeParser(
    buffer_size=1,
    breakpoint_percentile_threshold=95,
    embed_model=OpenAIEmbedding(),
)
```

หลักการ: วัด embedding similarity ระหว่างประโยท ถ้า similarity ลด → ตัด chunk ใหม่

เหมาะกับ: HR Policy, Technical Docs ที่แต่ละหัวข้อมีความหมายชัดเจน
ข้อดี: chunk มีความหมายครบ ไม่ตัดกลาง concept

---

### 3. Markdown Header / HTML Structure

แบ่งตามโครงสร้าง `#`, `##`, `###` — เหมาะกับ Markdown docs

**LangChain:**
```python
from langchain_text_splitters import MarkdownHeaderTextSplitter

splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=[
        ("#", "h1"),
        ("##", "h2"),
        ("###", "h3"),
    ]
)
# chunks[0].metadata = {"h1": "HR Policy", "h2": "วันลา"}
```

**LlamaIndex:**
```python
from llama_index.core.node_parser import MarkdownNodeParser

parser = MarkdownNodeParser.from_defaults()
```

เหมาะกับ: Technical documentation, README files, API docs

---

### 4. Hierarchical / Parent-Child (แนะนำ Advanced)

สร้าง chunks หลายระดับ — ค้นหาด้วย child เล็ก ๆ แต่ส่ง parent ใหญ่ให้ LLM

**LlamaIndex:**
```python
from llama_index.core.node_parser import HierarchicalNodeParser
from llama_index.core.retrievers import AutoMergingRetriever

# สร้าง 3 ระดับ: Document → Section → Sentence
parser = HierarchicalNodeParser.from_defaults(
    chunk_sizes=[2048, 512, 128]
)
# 2048 → Parent node (บริบทกว้าง ส่งให้ LLM)
# 512  → Child node (ค้นหา)
# 128  → Leaf node (precise match)

all_nodes = parser.get_nodes_from_documents(documents)

# AutoMergingRetriever: ถ้าดึง leaf nodes มาหลายอัน → auto merge เป็น parent
retriever = AutoMergingRetriever(base_retriever, storage_context)
```

เหมาะกับ: HR Policy (มีหมวดหมู่ชัดเจน), Onboarding, Product Docs
ข้อดี: ได้ทั้งความเร็ว (child) + context (parent)

---

### 5. Sentence Window

ค้นหาด้วยประโยคเดี่ยวเล็ก ๆ แต่เวลาส่งให้ LLM ขยาย context ด้วยประโยครอบข้าง

**LlamaIndex:**
```python
from llama_index.core.node_parser import SentenceWindowNodeParser

parser = SentenceWindowNodeParser.from_defaults(
    window_size=3,              # เก็บ 3 ประโยครอบข้างไว้ใน metadata
    window_metadata_key="window",
)

# Postprocessor: ขยาย context จาก window metadata
from llama_index.core.postprocessor import MetadataReplacementPostProcessor
postprocessor = MetadataReplacementPostProcessor(target_metadata_key="window")
```

เหมาะกับ: Q&A แบบ precise, คำถามที่ต้องการ context รอบข้าง เช่น "ถ้าเจ็บป่วยต้องทำยังไง?"

---

### 6. Code / JSON Splitter

สำหรับ source code หรือ JSON documents

**LangChain:**
```python
from langchain_text_splitters import (
    PythonCodeTextSplitter,
    RecursiveJsonSplitter,
)

code_splitter = PythonCodeTextSplitter(chunk_size=500, chunk_overlap=50)
json_splitter = RecursiveJsonSplitter(max_chunk_size=500)
```

---

### เปรียบเทียบสรุป

| Strategy | เหมาะกับ | ข้อดี | ข้อเสีย | Framework |
|---------|----------|--------|---------|-----------|
| Fixed-Size | เอกสารทั่วไป, เริ่มต้น | เร็ว, ง่าย | ตัด context ได้ | LC, LI |
| Semantic | HR Policy, Tech Docs | chunk มีความหมายครบ | ช้า (ต้อง embed) | LC, LI (TS ไม่มี) |
| Markdown Header | Documentation | preserve hierarchy | ใช้ได้เฉพาะ MD/HTML | LC, LI |
| Hierarchical | Docs ที่มี sections ชัดเจน | ได้ทั้ง precision + context | setup ซับซ้อน | LI (TS ไม่มี) |
| Sentence Window | Q&A precise, ขั้นตอน | เร็ว + context | เหมาะกับ short answers | LI |
| Code / JSON | Source code, APIs | แบ่งตาม structure | ตัวอย่างน้อย | LC |

---

### Chunking Strategy สำหรับ Sellsuki Docs

| เอกสาร | Strategy แนะนำ | เหตุผล |
|--------|---------------|--------|
| HR Policy (PDF) | Hierarchical | มีหมวดหมู่ชัดเจน, คำถามมักถามเรื่องเดียว |
| IT Guide | Sentence Window | ขั้นตอนต้องการ context รอบข้าง |
| Product Docs | Semantic | แต่ละ feature มีความหมายอิสระ |
| Dev Standards | Fixed-Size + overlap | Code snippets ต้องการ consistency |
| Onboarding | Hierarchical | มี sections ชัดเจน |

---

### Overlap — ทำไมต้องมี?

Overlap คือส่วนที่ซ้ำกันระหว่าง chunks ข้างเคียง — ช่วยไม่ให้ข้อมูลขาดระหว่างขอบ

```python
chunk_overlap=64,  # 64 characters/tokens
```

ถ้าไม่มี overlap: คำถาม "เงินเดือนออกวันไหน?" อาจโดนตัดจาก "เงินเดือน" กับ "ออกวันที่ 25" คนละ chunk → retrieval ไม่เจอ

---

### Metadata ที่ควรแนบทุก Chunk

```json
{
  "source_url": "https://docs.sellsuki.com/hr-policy",
  "doc_title": "HR Policy 2025",
  "section": "วันลา",
  "department": "hr",
  "doc_type": "policy",
  "access_level": "all",
  "last_updated": "2025-01-15",
  "language": "th",
  "chunk_index": 3,
  "parent_doc_id": "hr-policy-2025"
}
```

---

### ทดลองวัดผล

วิธีเดียวที่จะรู้ว่า strategy ไหนดีสุด → ทดลองแล้ววัด:

1. **Recall@5** — ได้ chunk ที่ถูกต้องอยู่ใน top 5 ไหม?
2. **MRR (Mean Reciprocal Rank)** — chunk ที่ถูกต้องอยู่ลำดับที่เท่าไหร่?
3. **Relevance** — LLM ตอบถูกต้องจาก chunk ที่ดึงมาไหม?

```
ทดลองทั้ง 4 แบบ แล้ววัด Recall / Precision
```

## เปรียบเทียบ LangChain vs LlamaIndex

| Feature | LangChain | LlamaIndex |
|---------|-----------|------------|
| Fixed-Size | ✅ `RecursiveCharacterTextSplitter` | ✅ `SentenceSplitter` |
| Semantic | ✅ `SemanticChunker` | ✅ `SemanticSplitterNodeParser` |
| Markdown Header | ✅ `MarkdownHeaderTextSplitter` | ✅ `MarkdownNodeParser` |
| Hierarchical | ❌ | ✅ `HierarchicalNodeParser` |
| Sentence Window | ❌ | ✅ `SentenceWindowNodeParser` |
| Code Splitter | ✅ `PythonCodeTextSplitter` | ✅ `CodeSplitter` |

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — Chunking คือขั้นตอนที่ 2 ใน Indexing Pipeline
- [[wiki/concepts/llamaindex-framework|LlamaIndex]] — มี NodeParsers หลากหลายกว่า
- [[wiki/concepts/langchain-framework|LangChain]] — มี TextSplitters พื้นฐานครบ

## แหล่งที่มา

- [[wiki/sources/rag-decision-guide|RAG Framework Decision Guide — Sellsuki]]
- [[wiki/sources/llamaindex-deep-dive|LlamaIndex & RAG Deep Dive]]
- [[wiki/sources/langchain-llamaindex-deep-dive|LangChain & LlamaIndex — Deep Dive Complete Guide]]
- [[wiki/sources/langchain-rag-guide|LangChain, RAG & Sellsuki Knowledge Base]]
