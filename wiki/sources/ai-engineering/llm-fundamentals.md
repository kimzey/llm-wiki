---
title: "LLM Fundamentals — พื้นฐาน Large Language Models"
type: source
source_file: "raw/notes/ai-engineering/02_llm_fundamentals.md"
tags: [llm, transformer, tokenization, temperature, training, embedding]
related: [wiki/concepts/llm-large-language-model, wiki/concepts/embedding, wiki/concepts/prompt-engineering]
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../../raw/notes/ai-engineering/02_llm_fundamentals.md|Original file]]

## สรุป

คู่มือพื้นฐาน LLM ครอบคลุมสถาปัตยกรรม Transformer, Tokenization, Temperature/Sampling Parameters, Training Stages (Pre-training → SFT → RLHF → Constitutional AI), Embeddings และการจัดการ Context Window อธิบายข้อจำกัดสำคัญเช่น Hallucination, Positional Bias, Sycophancy

## ประเด็นสำคัญ

- **LLM คือ Next Token Predictor** — ไม่ได้ "เข้าใจ" แต่เรียนรู้ pattern จากข้อมูลมหาศาล
- **Transformer Architecture**: Tokenizer → Embedding → Attention Layers × N → Feed-Forward → Output
- **Self-Attention**: กลไกที่ทำให้โมเดลรู้ว่าคำไหนเกี่ยวข้องกับคำไหน
- **Token ภาษาไทย**: 1 คำ ≈ 3-5 tokens → Context 200,000 tokens จริงใช้ได้ ~40,000-66,000 คำไทย
- **Temperature**: 0 = deterministic, 0.5 = สมดุล, 2.0 = สร้างสรรค์มาก
- **Top-P (Nucleus Sampling)**: เลือกจาก token ที่รวม probability ถึง P%
- **Training Stages**: Pre-training → SFT → RLHF → Constitutional AI (Anthropic)
- **Lost in the Middle**: โมเดลจำข้อมูลต้น-ท้าย context ดีกว่าตรงกลาง
- **Claude Family**: Haiku (เร็ว/ถูก), Sonnet (สมดุล), Opus (ฉลาดสุด)

## ข้อมูล / หลักฐาน ที่น่าสนใจ

> [!note] Training Stages
> 1. **Pre-training**: Internet ทั้งหมด หลาย Trillion tokens → Base Model
> 2. **SFT**: คู่ (คำถาม, คำตอบ) จากมนุษย์ → Instruct Model  
> 3. **RLHF**: มนุษย์จัดอันดับคำตอบ (A ดีกว่า B) → Assistant Model
> 4. **Constitutional AI** (Anthropic): โมเดลวิจารณ์ตัวเองตาม "Constitution"

```python
# นับ tokens ก่อนส่ง
client.messages.count_tokens(model="claude-sonnet-4-6", messages=[...])
```

- Sycophancy: โมเดลมีแนวโน้มเห็นด้วยกับ user เพื่อเอาใจ — แก้ด้วย System Prompt "อย่าเอาใจ"
- Context Poisoning: ถ้า RAG ดึงเอกสารผิด โมเดลตอบผิดตาม ("ขยะเข้า ขยะออก")
- Flash Attention: เทคนิคเพิ่มประสิทธิภาพ เร็วขึ้น 2-4x โดยไม่เสียความแม่นยำ

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/llm-large-language-model|LLM — Large Language Model]]
- [[wiki/concepts/embedding|Embedding — แปลงข้อความเป็น Vector]]
- [[wiki/concepts/prompt-engineering|Prompt Engineering]]
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]]
