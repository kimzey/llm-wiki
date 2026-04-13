---
title: "Haystack Deep Dive — Phase 1: รู้จัก Haystack & Core Concepts"
type: source
source_file: raw/notes/Hetstack/haystack-phase1-intro.md
url: ""
published: 2026-04-01
tags: [haystack, rag, pipeline, nlp, framework, python]
related: [wiki/sources/haystack-phase2-indexing.md, wiki/concepts/haystack-framework.md, wiki/concepts/rag-retrieval-augmented-generation.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/Hetstack/haystack-phase1-intro.md|Original file]]
> *ดูเพิ่ม: [[../../raw/notes/Hetstack/haystack-phase1-introduction.md|Introduction alt file]]*

## สรุป

Haystack คือ Open-Source Framework (by deepset) สำหรับสร้าง AI-powered Search & NLP Pipelines โดยเฉพาะ RAG, Question Answering, Semantic Search, และ Agent Systems — ใช้ Python เป็นหลัก สถาปัตยกรรมหลักคือ **Component → Pipeline → DocumentStore** ที่ทำงานแบบ DAG

## ประเด็นสำคัญ

- **Haystack 2.x** (ปัจจุบัน, `pip install haystack-ai`) — ต่างจาก 1.x มาก, API ใหม่ทั้งหมด, active development
- **Pipeline = DAG of Components**: add_component() + connect() เชื่อม output → input ระหว่าง component
- **Document** คือ unit พื้นฐาน: `content`, `meta`, `embedding`, `score`, `id` (auto-generated)
- **DocumentStore** คือฐานข้อมูลของ Haystack: InMemory (dev), Qdrant/Elasticsearch/Weaviate/Pinecone (production)
- **2 Pipeline หลัก**: Indexing Pipeline (ป้อนเอกสาร) + Query Pipeline (ตอบคำถาม)

## ข้อมูล / หลักฐาน ที่น่าสนใจ

**เปรียบเทียบ Haystack vs ทางเลือก:**
| Feature | Haystack | LangChain | LlamaIndex |
|---|---|---|---|
| Focus | Production-ready Pipelines | General LLM Apps | Data Indexing |
| Pipeline Validation | ✅ Schema-based | ❌ | ❌ |
| Learning Curve | ปานกลาง | ต่ำ | ปานกลาง |

**Component types ที่ใช้บ่อย:**
- `Converter`: PyPDFToDocument, HTMLToDocument
- `Preprocessor`: DocumentSplitter, DocumentCleaner
- `Embedder`: SentenceTransformersDocumentEmbedder
- `Retriever`: InMemoryBM25Retriever, QdrantEmbeddingRetriever
- `Generator`: OpenAIGenerator, HuggingFaceLocalGenerator
- `Builder`: PromptBuilder (Jinja2 templating)

**Hello World RAG pattern:**
```
DocumentStore → BM25Retriever → PromptBuilder → OpenAIGenerator
```

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่มีข้อขัดแย้ง — เพิ่มเติมรายละเอียด implementation ของ Haystack framework

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/haystack-framework.md|Haystack Framework]]
- [[wiki/concepts/rag-retrieval-augmented-generation.md|RAG — Retrieval-Augmented Generation]]
