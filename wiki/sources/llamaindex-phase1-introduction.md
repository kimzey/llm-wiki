---
title: "LlamaIndex Phase 1 — Introduction & Core Concepts"
type: source
source_file: raw/notes/liamaindex/llamaindex-phase1-introduction.md
tags: [llamaindex, rag, tutorial, introduction]
related: [wiki/concepts/llamaindex-framework.md, wiki/concepts/rag-retrieval-augmented-generation.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/liamaindex/llamaindex-phase1-introduction.md|Original file]]

## สรุป

บทช่วยสอน LlamaIndex Phase 1: แนะนำ LlamaIndex framework และ concepts พื้นฐาน — LLMs มี knowledge cutoff และไม่รู้จักข้อมูลส่วนตัว, LlamaIndex สร้าง bridge ระหว่างข้อมูลเหล่านี้กับ LLM ด้วยเทคนิค RAG, เปรียบเทียบ LlamaIndex vs LangChain, ติดตั้ง, ใช้งาน Documents & Nodes, Settings object, สร้าง RAG ง่ายๆ ด้วย VectorStoreIndex, และ Query Engine

## ประเด็นสำคัญ

- **LlamaIndex vs LangChain**: LlamaIndex เน้น data indexing & retrieval (RAGลึก), LangChain เน้น general-purpose orchestration (Chains, Agents)
- **Architecture 5 Building Blocks**: Documents & Nodes (หน่วยข้อมูล), Loaders (อ่านข้อมูล), Indexes (จัดเก็บ), Query Engines (ตอบคำถาม), LLMs & Embeddings (โมเดล)
- **Document vs Node**: Document = หน่วยข้อมูลหลัก (ทั้งไฟล์), Node = ชิ้นส่วนย่อย (chunk) หลังแบ่ง
- **Settings object (v0.10+)**: global config สำหรับ LLM + Embeddings model, chunk_size, context_window — แทน ServiceContext (deprecated)
- **RAG Pipeline 4 บรรทัด**: `VectorStoreIndex.from_documents(documents)` → `as_query_engine()` → `.query("คำถาม")` เสร็จ
- **Query Pipeline**: Embed Query → Retrieve (Top-K nodes) → Synthesize (LLM) → Response (พร้อม source nodes)
- **Streaming Response**: รองรับ streaming tokens ด้วย `streaming=True`
- **Prompt Templates**: customize QA prompts สำหรับภาษาไทยได้

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- **Hello World RAG 4 บรรทัด**:
  ```python
  documents = SimpleDirectoryReader("./data").load_data()
  index = VectorStoreIndex.from_documents(documents)
  query_engine = index.as_query_engine()
  response = query_engine.query("สรุปเนื้อหาหลัก")
  ```

- **Response Object**: มีทั้งข้อความตอบ (`response.response`), source nodes (`response.source_nodes`), และ raw LLM response (`response.raw`)

- **Settings configuration**:
  ```python
  Settings.llm = OpenAI(model="gpt-4o", temperature=0.1)
  Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")
  Settings.chunk_size = 1024
  Settings.chunk_overlap = 20
  ```

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

- LlamaIndex v0.10+ ใช้ `Settings` object แทน `ServiceContext` (deprecated) — ต้องอัปเดต code เก่า

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/llamaindex-framework|LlamaIndex Framework]]
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG — Retrieval-Augmented Generation]]
- [[wiki/concepts/llm-large-language-model|LLM — Large Language Model]]
