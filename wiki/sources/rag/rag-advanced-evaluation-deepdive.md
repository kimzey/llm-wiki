---
title: "Advanced Evaluation Deep Dive — LLM-as-Judge"
type: source
source_file: raw/notes/LangChain/advanced-evaluation-deep-dive.md
url: ""
published: 2026-01-01
tags: [evaluation, llm-judge, langsmith, ragas, rag-triad, pairwise-comparison, agent-evaluation, ci-cd]
related: [wiki/concepts/rag-evaluation.md, wiki/concepts/langchain-framework.md, wiki/concepts/agentic-rag.md]
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../../raw/notes/langchain/advanced-evaluation-deep-dive.md|Original file]]


## สรุป

คู่มือ Advanced Evaluation ที่เน้น LLM-as-Judge pattern — เมื่อ rule-based evaluator ไม่เพียงพอให้ใช้ LLM ตัดสินคุณภาพ — ครอบคลุม Single-Point Grading, Multi-Dimension Grading (4 มิติใน 1 call), Pairwise Comparison, Reference-based Judge, RAG-specific Evaluators (RAG Triad), Agent-specific Evaluators (tool selection + task completion), LangSmith integration แบบ end-to-end, Evaluation Pipeline อัตโนมัติสำหรับ CI/CD, และ Pitfalls/Best Practices

## ประเด็นสำคัญ

- **LLM-as-Judge vs Rule-based**: Rule-based เหมาะ exact match/format check, LLM-as-Judge เหมาะคุณภาพการเขียน/relevance/hallucination/tone
- **Judge Prompt Design**: ต้องมี Role + Criteria + Scale (rubric คำอธิบายแต่ละระดับ) + Input + Output format — อย่าถามกว้างๆ เช่น "ดีไหม?"
- **Single-Point Grading**: ตรวจมิติเดียว เช่น relevance, helpfulness, tone — return `{"key": "...", "score": 0-1, "comment": "..."}`
- **Multi-Dimension Grading**: ให้คะแนน 4 มิติ (relevance/completeness/clarity/accuracy) ใน 1 LLM call — ประหยัดค่าใช้จ่าย
- **Pairwise Comparison**: เปรียบเทียบ 2 versions — สลับตำแหน่ง A/B เพื่อลด Position Bias
- **RAG Triad**: Context Relevance + Faithfulness + Answer Relevance — 3 มิติหลักของ RAG quality
- **Agent Evaluators**: Tool Selection (exact match) + Task Completion (LLM-as-Judge)
- **LangSmith Pipeline**: `evaluate(bot_fn, data="dataset-name", evaluators=[...])` — ดู dashboard ออนไลน์
- **CI/CD Integration**: รัน eval script ใน pipeline — exit code 0=pass, 1=fail พร้อม threshold 0.7

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- `parse_judge_response()` helper: parse `SCORE:` และ `REASON:` จาก response — handle "4/5" format ด้วย
- Factory function `create_judge_evaluator()`: สร้าง evaluator reusable ด้วย criteria prompt
- `multi_dimension_judge()` ใน 1 LLM call ประหยัด vs รัน 4 evaluators แยก = เสีย 4x call
- สำหรับ Pairwise: random สลับตำแหน่ง A/B ทุกครั้งเพื่อลด position bias — แต่ต้อง map กลับ label จริง
- `max_concurrency=4` ใน `evaluate()` รันหลาย test cases พร้อมกัน — เร็วขึ้น 4x

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่มีข้อขัดแย้ง — เป็น advanced detail ที่เพิ่มเติมจาก RAG Evaluation knowledge เดิม เนื้อหาสอดคล้องกับ RAGAS pattern

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-evaluation|RAG Evaluation]] — concept หลักที่ document นี้ขยายความ
- [[wiki/concepts/langchain-framework|LangChain Framework]] — ใช้ LangChain + LangSmith สำหรับ evaluation pipeline
- [[wiki/concepts/agentic-rag|Agentic RAG]] — Agent-specific evaluators ใน Section 8
