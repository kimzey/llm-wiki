---
title: "Evaluation & Testing — การวัดคุณภาพ AI"
type: source
source_file: "raw/notes/ai-engineering/06_evaluation_testing.md"
tags: [evaluation, testing, bleu, rouge, llm-as-judge, a-b-testing, regression]
related: [wiki/concepts/llm-evaluation, wiki/concepts/rag-evaluation, wiki/concepts/prompt-engineering]
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../../raw/notes/ai-engineering/06_evaluation_testing.md|Original file]]

## สรุป

คู่มือ Evaluation & Testing ครอบคลุม 3 วิธีประเมิน (Automated Metrics, LLM-as-Judge, Human Evaluation), Golden Dataset, Regression Testing, A/B Testing (Shadow Mode + Canary), Production Monitoring พร้อม Anomaly Detection และ Safety Evaluation ด้วย Red Teaming

## ประเด็นสำคัญ

- **BLEU**: วัดการซ้อนทับของ n-gram — ง่าย แต่ไม่แม่นยำ (ความหมายเหมือนกันแต่คำต่าง → score ต่ำ)
- **ROUGE**: นิยมสำหรับ Summarization
- **BERTScore**: วัดด้วย semantic similarity — แม่นยำกว่า
- **LLM-as-Judge**: ใช้ LLM ประเมิน 4 มิติ — Accuracy, Completeness, Clarity, Relevance (1-5)
- **Golden Dataset**: ชุดทดสอบมาตรฐาน — แยกตาม category และ difficulty
- **Regression Testing**: ทดสอบทุกครั้งที่เปลี่ยน prompt/model — บล็อก deploy ถ้า score ลดเกิน 10%
- **Canary Deployment**: 5% traffic → ถ้าดีขึ้น → เพิ่ม traffic → 100%
- **Shadow Testing**: รัน Model B ซ่อน ๆ โดยไม่ส่งผลให้ user
- **Safety Metrics**: Prompt Injection, Data Leakage, Bias, Harmful Content

## ข้อมูล / หลักฐาน ที่น่าสนใจ

> [!note] LLM-as-Judge Template
> ```python
> JUDGE_PROMPT = """ประเมิน 1-5:
> 1. ความถูกต้อง (Accuracy)
> 2. ความครบถ้วน (Completeness)  
> 3. ความชัดเจน (Clarity)
> 4. ความเกี่ยวข้อง (Relevance)
> ตอบ JSON: {"accuracy": X, ...}"""
> ```

- Pre-deployment Checklist: Functional (pass rate >95%) + Safety + Performance (p95 <3s) + Business Metrics
- CI/CD Integration: รัน eval suite ทุก PR ที่แตะ `prompts/**` หรือ `config/model.yaml`
- Business Metrics สำคัญ: task_completion_rate, escalation_rate, retry_rate (ยิ่งต่ำยิ่งดี)

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/llm-evaluation|การประเมิน LLM]]
- [[wiki/concepts/rag-evaluation|RAG Evaluation]]
- [[wiki/concepts/prompt-engineering|Prompt Engineering]]
