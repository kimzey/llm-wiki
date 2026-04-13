---
title: "RAG คู่มือฉบับสมบูรณ์ — ตั้งแต่พื้นฐานจนถึง Production"
type: source
source_file: raw/notes/rag-knowledge/rag-complete-guide.md
url: ""
published: 2026-03-01
tags: [rag, chunking, embedding, retrieval, evaluation, ragas, llama-index, langchain, haystack, deployment]
related: [wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/rag-chunking-strategies.md, wiki/concepts/rag-evaluation.md, wiki/concepts/langchain-framework.md, wiki/concepts/llamaindex-framework.md, wiki/concepts/hybrid-search-bm25-vector.md]
created: 2026-04-14
updated: 2026-04-14
---

**Full source**: [[../../raw/notes/rag-knowledge/rag-complete-guide.md|Original file]]

## สรุป

คู่มือ RAG ครอบคลุมครบทุกด้าน: RAG pipeline 8 ขั้นตอน, chunking strategies 7 แบบ, embedding models, vector stores, retrieval methods, การเปรียบเทียบ frameworks (LangChain, LlamaIndex, Haystack), deployment options, evaluation ด้วย RAGAS และแนะนำตามสถานการณ์

สร้างจากการสนทนากับ Claude มีนาคม 2026

## ประเด็นสำคัญ

### RAG Pipeline 8 ขั้นตอน

```
เอกสาร → Load → Chunk → Embed → Store → Retrieve → Augment → Generate → คำตอบ
```

| ขั้นตอน | ทำอะไร | ตัวอย่าง |
|--------|--------|---------|
| Load | โหลดข้อมูล | PDF, Word, Notion, DB |
| Chunk | ตัดเป็นชิ้นเล็ก | ตามประโยค, ตามความหมาย |
| Embed | แปลงเป็น vector | OpenAI embedding, BGE-M3 |
| Store | เก็บใน vector DB | ChromaDB, Pinecone, pgvector |
| Retrieve | ค้นข้อมูลที่เกี่ยวข้อง | Vector search, Hybrid search |
| Augment | ประกอบ prompt | ใส่ context + คำถาม |
| Generate | สร้างคำตอบ | GPT-4, Claude, Gemini |
| Evaluate | วัดคุณภาพ | RAGAS framework |

### Chunking Strategies ทั้ง 7 แบบ (เรียงตามความซับซ้อน)

```
ง่าย/เร็ว ←─────────────────────────────→ แม่น/ช้า

Fixed → Recursive → Sentence → Structure → Semantic → Hierarchical → Agentic
  ↑                    ↑                       ↑                         ↑
  prototype         ส่วนใหญ่ใช้            คุณภาพสูง                แพงสุด
```

1. **Fixed Size**: ตัดตาม characters/tokens ที่กำหนด — ง่ายสุด เร็วสุด แต่ตัดกลางประโยคได้
2. **Sentence**: ตัดตามประโยค — ไม่ตัดกลางประโยค **นิยมที่สุดสำหรับงานทั่วไป**
3. **Recursive**: ลองตัดที่ `\n\n` → `\n` → `.` → space — LangChain default รักษาโครงสร้างดี
4. **Semantic**: ตัดเมื่อความหมายเปลี่ยน (embedding) — แม่นที่สุด แต่ช้าที่สุด
5. **Hierarchical (Parent-Child)**: ค้นด้วย child เล็ก → ส่ง parent ใหญ่ให้ LLM
6. **Structure-Based**: ตาม Markdown header, HTML tag, code structure, PDF page
7. **Agentic**: ให้ LLM ตัดให้ — แม่นที่สุด แพงที่สุด ช้าที่สุด

### Retrieval Methods

```
แม่นน้อย ←──────────────────────────→ แม่นมาก

Vector only → + BM25 (Hybrid) → + Reranking → + Multi-Query
    ↑              ↑                  ↑
    เริ่มต้น       production         คุณภาพสูงสุด
```

- **Vector Search**: เข้าใจความหมาย แต่อาจพลาด exact keyword
- **BM25 (Keyword)**: แม่นกับชื่อเฉพาะ รหัส ตัวเลข แต่ไม่เข้าใจ synonym
- **Hybrid (แนะนำ)**: รวมทั้งสอง + Reciprocal Rank Fusion
- **Reranking**: ดึงมา 20 chunks → cross-encoder ให้คะแนน → top 3 ส่ง LLM
- **Multi-Query**: LLM สร้างคำถามหลายรูปแบบ → ค้นทุกรูปแบบ → รวมผล
- **Metadata Filtering**: กรองด้วย filter (category, year ฯลฯ) ก่อน vector search

### Framework Comparison

| ด้าน | LangChain | LlamaIndex | Haystack |
|------|-----------|------------|---------|
| ปรัชญา | ทำได้ทุกอย่าง | ทำให้ง่ายที่สุด | ควบคุมทุก step |
| Code เริ่มต้น | ~20 บรรทัด | ~5 บรรทัด | ~30-50 บรรทัด |
| Production readiness | ดี | ต้องจัดการเอง | ออกแบบมาเพื่อ production |
| API Server built-in | ✅ LangServe | FastAPI เอง | ✅ Hayhooks |
| TypeScript support | ✅ LangChain.js | ✅ (ฟีเจอร์น้อยกว่า) | ❌ Python only |
| Community | ใหญ่สุด | ใหญ่ | enterprise เยอะ |

### RAG Evaluation (RAGAS)

| ตัววัด | วัดอะไร |
|--------|--------|
| **Faithfulness** | คำตอบตรงกับ source ไหม? |
| **Answer Relevancy** | คำตอบตรงคำถามไหม? |
| **Context Precision** | ดึง context ถูกต้องไหม? |
| **Context Recall** | ดึง context มาครบไหม? |

### Deployment Options

- **No-Code**: OpenAI GPTs, Chatbase, CustomGPT — ได้ใน 30 นาที
- **Low-Code**: Dify, Flowise, Langflow, Streamlit — 2-3 ชั่วโมง
- **Framework**: LangChain + LangServe, LlamaIndex + FastAPI — 1-3 วัน
- **เขียนเอง**: FastAPI + ChromaDB + OpenAI — 3-7 วัน

**Cloud Run (Singapore) หรือ AWS ECS** สำหรับ production ไทย

### แนวทางแนะนำสำหรับ Beginners

1. Sentence chunking
2. text-embedding-3-small
3. ChromaDB
4. Vector search
5. GPT-4o-mini

→ ทำได้ใน 30 นาที แล้วค่อยปรับทีละขั้น — คุณภาพจะดีขึ้นมากที่สุดจากการปรับ chunking และเพิ่ม hybrid search + reranking

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- LlamaIndex มี NodeParsers ครบสุด: SentenceSplitter, SemanticSplitter, HierarchicalNodeParser, SentenceWindowNodeParser, MarkdownNodeParser, HTMLNodeParser, CodeSplitter
- Haystack ใช้ Component-based architecture ทุกอย่างเป็น Component ต่อกันเป็น Pipeline — debug ง่ายเพราะเห็น flow ชัด
- LangChain.js (TypeScript) มีฟีเจอร์ครบสำหรับ web/mobile frontend

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบความขัดแย้ง — เนื้อหาสอดคล้องกับ wiki ที่มีอยู่แล้ว

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]]
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]]
- [[wiki/concepts/rag-evaluation|RAG Evaluation]]
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search BM25 + Vector]]
- [[wiki/concepts/langchain-framework|LangChain Framework]]
- [[wiki/concepts/llamaindex-framework|LlamaIndex Framework]]
- [[wiki/concepts/haystack-framework|Haystack Framework]]
