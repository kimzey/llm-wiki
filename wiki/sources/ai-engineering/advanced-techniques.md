---
title: "Advanced Techniques — เทคนิคขั้นสูง AI Engineering"
type: source
source_file: "raw/notes/ai-engineering/08_advanced_techniques.md"
tags: [meta-prompting, apo, mixture-of-agents, structured-output, graphrag, speculative-decoding]
related: [wiki/concepts/prompt-engineering, wiki/concepts/ai-agent, wiki/concepts/rag-retrieval-augmented-generation]
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../../raw/notes/ai-engineering/08_advanced_techniques.md|Original file]]

## สรุป

คู่มือเทคนิคขั้นสูงครอบคลุม Meta-Prompting (LLM เขียน prompt ให้ตัวเอง), Automatic Prompt Optimization (APO), Constitutional AI Prompting, Advanced RAG (GraphRAG, RAPTOR, Adaptive RAG, Multi-hop), Mixture of Agents (MoA), Structured Output, Performance Optimization (Batching, Prompt Compression, KV Cache), Multimodal, และ Prompt Versioning

## ประเด็นสำคัญ

- **Meta-Prompting**: ใช้ LLM สร้าง prompt ที่ดีที่สุดสำหรับงาน — automated prompt design
- **APO (Automatic Prompt Optimization)**: วนรอบ evaluate → วิเคราะห์ failure → แก้ prompt → ทดสอบใหม่
- **GraphRAG**: เก็บความรู้เป็น Knowledge Graph — ค้นหาผ่าน relationship แทน keyword
- **RAPTOR**: สร้าง Tree of Summaries หลายระดับ — ค้นได้ทั้งรายละเอียดและภาพรวม
- **Adaptive RAG**: จัดระดับ complexity ก่อน (simple/factual/complex/multi-hop) → เลือก strategy
- **Mixture of Agents (MoA)**: หลายโมเดลตอบพร้อมกัน → Aggregator รวมผล — ลด variance
- **Speculative Decoding**: โมเดลเล็ก draft → โมเดลใหญ่ verify — เร็วขึ้น 2-3x
- **KV Cache Optimization**: วาง static content (long document) ต้น → dynamic query ท้าย
- **Prompt Versioning**: จัดการ prompt เหมือน code — registry + version + A/B

## ข้อมูล / หลักฐาน ที่น่าสนใจ

> [!tip] KV Cache Pattern
> ```python
> # GOOD: static content ต้น → cache ได้!
> prompt = f"{long_document}\n\nคำถาม: {user_question}"
> # BAD: static content ท้าย → cache ไม่ได้
> prompt = f"คำถาม: {user_question}\n\n{long_document}"
> ```

- Batching: ใส่หลาย text ใน 1 prompt หรือ parallel calls ด้วย asyncio.Semaphore
- Prompt Compression: ลด prompt 30% โดยไม่เสียข้อมูลสำคัญ ด้วย LLM วิเคราะห์และตัด
- Agentic Workflow with Interruption: Agent pause แสดงแผนก่อน → รอ human approval → execute

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/prompt-engineering|Prompt Engineering]]
- [[wiki/concepts/ai-agent|AI Agent]]
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]]
- [[wiki/concepts/agentic-rag|Agentic RAG]]
