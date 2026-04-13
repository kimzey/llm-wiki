---
title: "Sellsuki RAG Agent Guide (06)"
type: source
source_file: raw/notes/rag-knowledge/06-sellsuki-agent-guide.md
url: ""
published: 2026-01-01
tags: [rag, sellsuki, pgvector, agent, implementation, langchain, llamaindex]
related: [wiki/sources/sellsuki-rag-complete-guide.md, wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/pgvector.md]
created: 2026-04-14
updated: 2026-04-14
---

**Full source**: [[../../raw/notes/rag-knowledge/06-sellsuki-agent-guide.md|Original file]]

## สรุป

ไฟล์นี้มีเนื้อหาเหมือนกันกับ [[wiki/sources/sellsuki-rag-complete-guide|Sellsuki RAG Agent - Complete Implementation Guide]] ทุกประการ — เป็นสำเนาของ guide เดียวกันที่บันทึกไว้ในชื่อไฟล์ต่างกัน (`06-sellsuki-agent-guide.md`)

เนื้อหาครอบคลุม:
- Phase 1: เตรียม Data (extractors สำหรับ PDF, Word, HTML, CSV, Notion)
- Phase 2: Chunking & Embedding (Recursive, Semantic, Markdown Header)
- Phase 3: Vector Database ด้วย pgvector
- Phase 4: เปรียบเทียบ RAG Frameworks (LangChain, LlamaIndex, Google ADK, CrewAI, DIY)
- Phase 5: สร้าง Agent พร้อม FastAPI
- Phase 6: Deployment (Cloud Run, Docker Compose, VPS)
- Implementation Plan (6 สัปดาห์)

## ประเด็นสำคัญ

ดูรายละเอียดทั้งหมดที่ [[wiki/sources/sellsuki-rag-complete-guide|Sellsuki RAG Agent - Complete Implementation Guide]]

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]]
- [[wiki/concepts/pgvector|pgvector]]
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]]
- [[wiki/concepts/langchain-framework|LangChain Framework]]
- [[wiki/concepts/agentic-rag|Agentic RAG]]
