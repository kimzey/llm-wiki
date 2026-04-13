---
title: "Langflow — Visual AI Workflow Builder"
type: concept
tags: [langflow, visual-workflow, pipeline, no-code, rag, ai-agent, drag-and-drop]
sources: [phase4-langflow.md, openrag-backend-vs-langflow.md]
related: [wiki/concepts/openrag-platform.md, wiki/concepts/ai-agents-system.md, wiki/concepts/langchain-framework.md]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น
Langflow เป็น Open-Source Visual Workflow Builder สำหรับสร้าง AI Pipelines แบบ Drag-and-Drop โดยไม่ต้องเขียนโค้ดมาก — เชื่อม components (LLM, embeddings, vector stores, tools) ด้วย visual edges และ export เป็น API ได้

## อธิบาย
Langflow ทำงานแบบ flow-based:
- **Node** = component แต่ละอัน (Docling, Text Splitter, OpenAI, OpenSearch)
- **Edge** = การเชื่อมต่อระหว่าง components
- **Flow** = pipeline ทั้งหมด — export เป็น JSON + call ผ่าน API

ใน OpenRAG, Langflow ทำหน้าที่ **AI Pipeline Engine** ล้วนๆ — ไม่รู้จัก user, ไม่จัดการ auth รับแค่ metadata จาก Backend ผ่าน HTTP headers และประมวลผล

## ประเด็นสำคัญ
- **Modular & Configurable**: เปลี่ยน LLM provider ได้โดยแก้ที่ UI ไม่ต้องแก้โค้ด
- **API Export**: ทุก Flow เรียกได้ผ่าน `POST /api/v1/run/{flow_id}`
- **Streaming**: รองรับ SSE streaming สำหรับ chat
- **Custom Components**: เขียน Python class ใหม่ได้
- **MCP Integration**: เชื่อม external tools ผ่าน MCP nodes
- **Built on LangChain**: ใช้ LangChain components ภายใต้ hood

**ข้อดีกว่าเขียนโค้ดตรง:**
- เปลี่ยน pipeline (เช่น เพิ่ม reranker) ได้โดยลาก node ใหม่
- ทดสอบ flow ใน UI ได้โดยตรง
- Backup/restore flow เป็น JSON

## ตัวอย่าง / กรณีศึกษา

**Ingestion Flow ใน OpenRAG:**
```
[File Input] → [Docling (parse)] → [SplitText (chunk_size=1000)] →
[OpenAIEmbeddings] → [OpenSearchVectorStore (index=documents)]
```

**Chat Flow:**
```
[ChatInput] → [Agent (LLM)] ←→ [OpenSearchRetriever (KNN+DLS)] → [ChatOutput]
```

## ความสัมพันธ์กับ concept อื่น
- [[wiki/concepts/openrag-platform.md|OpenRAG]] — ใช้ Langflow เป็น AI pipeline engine
- [[wiki/concepts/langchain-framework.md|LangChain]] — Langflow built on LangChain components
- [[wiki/concepts/ai-agents-system.md|AI Agents]] — Langflow รองรับ Agent nodes

## แหล่งที่มา
- phase4-langflow.md (OpenRAG Phase 4 documentation)
- openrag-backend-vs-langflow.md (Architecture separation)
