---
title: "การประเมิน LLM (Evaluation & Testing)"
type: concept
tags: [evaluation, testing, bleu, rouge, llm-as-judge, a-b-testing, regression, monitoring]
sources: [raw/notes/ai-engineering/06_evaluation_testing.md]
related: [wiki/concepts/rag-evaluation, wiki/concepts/prompt-engineering, wiki/concepts/ai-production-deployment]
created: 2026-04-14
updated: 2026-04-14
---

## สรุปสั้น

LLM Evaluation คือกระบวนการวัดคุณภาพระบบ AI อย่างเป็นระบบ ตั้งแต่ Automated Metrics (BLEU/ROUGE/BERTScore), LLM-as-Judge, Human Evaluation จนถึง Regression Testing และ Production Monitoring

## อธิบาย

### ทำไม Evaluation ถึงสำคัญ

ปัญหาที่พบบ่อยโดยไม่มี evaluation:
- "โมเดลดูดีในการทดสอบ แต่พัง production"
- "อัปเดต prompt แล้วพัง use case เก่า"
- "ไม่รู้ว่า version ไหนดีกว่า"

### 3 วิธีการประเมินหลัก

#### 1. Automated Metrics

**String-based** (ง่าย แต่ไม่แม่นยำ):
- **BLEU**: วัดการซ้อนทับของ n-gram — "แมวกินปลา" vs "ปลาถูกแมวกิน" → score ต่ำ แต่ความหมายเหมือนกัน
- **ROUGE**: นิยมสำหรับ Summarization

**Embedding-based** (ดีกว่า):
- **BERTScore**: วัดด้วย semantic similarity แทนการตรงตัวอักษร

#### 2. LLM-as-Judge (แนะนำ)
ใช้ LLM อีกตัวประเมิน 4 มิติ (1-5):
1. **Accuracy** — ข้อมูลถูกต้องไหม
2. **Completeness** — ตอบครบทุกส่วนไหม
3. **Clarity** — เข้าใจง่ายไหม
4. **Relevance** — ตรงคำถามไหม

#### 3. Human Evaluation
- **Crowdsourcing**: เร็ว ถูก แต่คุณภาพ annotator ไม่แน่นอน
- **Expert Evaluation**: ช้า แพง แต่แม่นยำ — เหมาะสำหรับ medical, legal
- **Preference Testing**: A vs B — Blind test, วัด win rate

### Regression Testing

ทดสอบทุกครั้งที่เปลี่ยน prompt หรือ model:
- บล็อก deployment ถ้า score ลดลงเกิน 10% ใน test cases ใด
- Integrate กับ CI/CD — รัน eval suite ทุก PR ที่แตะ `prompts/**`

### A/B Testing

**Shadow Mode**: รัน Model B ซ่อนๆ ไม่ส่งผลให้ user → เปรียบเทียบผล A vs B

**Canary Deployment**:
```
95% traffic → Model A (stable)
 5% traffic → Model B (new)
→ ถ้า B ดีกว่า เพิ่ม traffic ทีละน้อย → 100%
```

## ประเด็นสำคัญ

- **Golden Dataset**: ชุดทดสอบมาตรฐาน แยกตาม category + difficulty (easy/medium/hard)
- **Production Metrics**: latency_p95, hallucination_rate, task_success_rate, cost_per_request
- **Safety Evaluation**: Red Teaming — พยายาม Jailbreak, Prompt Injection, Data Extraction, Bias Testing
- **Pre-deployment Checklist**: Functional (>95%) + Safety + Performance (p95 <3s) + Business Metrics

## ตัวอย่าง / กรณีศึกษา

```python
# LLM Judge
JUDGE_PROMPT = """ประเมินคำตอบ (1-5) 4 มิติ:
1. Accuracy, 2. Completeness, 3. Clarity, 4. Relevance
ตอบ JSON: {"accuracy": X, "completeness": X, ...}"""

# Online Business Metrics ที่ track
metrics = {
    "task_completion_rate": "% ที่ user ทำงานสำเร็จ",
    "escalation_rate": "% ที่ต้องโอนให้มนุษย์",
    "retry_rate": "% ที่ user ถามซ้ำ (โมเดลตอบไม่ตรง)",
}
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/rag-evaluation|RAG Evaluation]] — Evaluation เฉพาะสำหรับ RAG System (Faithfulness, Context Relevance)
- [[wiki/concepts/prompt-engineering|Prompt Engineering]] — Evaluation วัดว่า prompt ดีแค่ไหน
- [[wiki/concepts/ai-production-deployment|AI Production Deployment]] — Production Monitoring เป็นส่วนหนึ่งของ Evaluation

## แหล่งที่มา

- [[wiki/sources/ai-engineering/evaluation-testing|Evaluation & Testing — การวัดคุณภาพ AI]]
