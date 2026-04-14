---
title: "Context & Prompt Engineering — คู่มือฉบับสมบูรณ์"
type: source
source_file: "raw/notes/ai-engineering/AI_Engineering_Complete_Guide.md"
tags: [context-engineering, prompt-engineering, llm, context-window, rag]
related: [wiki/concepts/context-engineering, wiki/concepts/prompt-engineering, wiki/concepts/llm-large-language-model]
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../../raw/notes/ai-engineering/AI_Engineering_Complete_Guide.md|Original file]]

## สรุป

คู่มือครอบคลุมทั้ง Context Engineering และ Prompt Engineering ตั้งแต่พื้นฐาน LLM, Context Window, Token จนถึงเทคนิค Advanced เช่น CoT, ReAct, Function Calling แยกแยะชัดเจนว่า Context Engineering (จัดการข้อมูลใน context) ต่างจาก Prompt Engineering (วิธีเขียนคำสั่ง) อย่างไร

## ประเด็นสำคัญ

- **Context Window** คือโต๊ะทำงานของโมเดล — ยิ่งใหญ่ยิ่งวางของได้มาก แต่ราคาแพงขึ้น
- **Context Engineering** = ศาสตร์การจัดการข้อมูลใน context ให้ครบถ้วนและมีประสิทธิภาพ
- **Prompt Engineering** = ศาสตร์การเขียนคำสั่งให้โมเดลตอบได้ผลลัพธ์ที่ต้องการ
- **4 เสาหลักของ Context Engineering**: External Memory, RAG + Dynamic Filters, Context Compaction, Context Isolation
- **Prompt Patterns**: Zero-shot, Few-shot, Chain-of-Thought, ReAct, Role Prompting, Constraint Prompting
- **Token ภาษาไทย**: 1 คำ ≈ 3-5 tokens (แพงกว่าภาษาอังกฤษ 3 เท่า)
- **Context Compaction**: Summarization → Sliding Window → Token Pruning → Hierarchical Memory
- **Context Isolation**: แยก trusted zone (system prompt) กับ untrusted zone (user input / external data)

## ข้อมูล / หลักฐาน ที่น่าสนใจ

> [!note] Context Engineering vs Prompt Engineering
> ถ้า Prompt Engineering = วิธีถามคำถาม, Context Engineering = วิธีจัดเตรียมข้อมูลทั้งหมดที่โมเดลต้องการก่อนตอบ

- Multi-tenant RAG: ต้องแยก namespace ใน Vector DB ด้วย `tenant_id` ป้องกัน data leakage
- CoT (Chain-of-Thought): แค่เพิ่ม "คิดทีละขั้นตอน" ใน prompt ช่วยให้โมเดลทำโจทย์ซับซ้อนได้ดีขึ้น
- Tree of Thought (ToT): คิดหลายสาขาพร้อมกันแล้วเลือกทางที่ดีสุด — ดีกว่า CoT สำหรับปัญหาซับซ้อน
- Function Calling: โมเดลตัดสินใจเองว่าจะเรียก function ไหน ไม่ใช่แค่ถาม-ตอบ

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/context-engineering|Context Engineering]]
- [[wiki/concepts/prompt-engineering|Prompt Engineering]]
- [[wiki/concepts/llm-large-language-model|LLM — Large Language Model]]
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]]
- [[wiki/concepts/ai-agent|AI Agent]]
