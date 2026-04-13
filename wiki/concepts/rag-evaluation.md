---
title: "RAG Evaluation"
type: concept
tags: [rag, evaluation, faithfulness, ragas, metrics, llm-judge, context-relevance]
sources: [haystack-phase5-custom-production.md, haystack-phase5-integrations.md, haystack-phase6-api-auth-observability.md]
related: [wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/haystack-framework.md, wiki/concepts/semantic-caching.md]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

**RAG Evaluation** คือกระบวนการวัดคุณภาพของ RAG Pipeline ใน 3 มิติ — Retrieval quality (ดึงเอกสารถูกต้องไหม), Generation quality (คำตอบ faithful กับ context ไหม), และ End-to-end quality (คำตอบตรงกับ ground truth ไหม) — เครื่องมือหลักคือ built-in Haystack evaluators และ RAGAS

## อธิบาย

### 3 Layer ของ RAG Evaluation

```
┌────────────────────────────────────────────────────────┐
│              RAG Evaluation Metrics                    │
├────────────────┬───────────────────────────────────────┤
│ Retrieval      │ Recall@K, Precision@K, MRR            │
│                │ Context Relevance                     │
├────────────────┼───────────────────────────────────────┤
│ Generation     │ Faithfulness, Answer Relevance         │
├────────────────┼───────────────────────────────────────┤
│ End-to-end     │ Exact Match, F1, BLEU, SAS             │
└────────────────┴───────────────────────────────────────┘
```

### Metrics สำคัญ

**Faithfulness** — คำตอบ LLM อ้างอิงจาก context จริงๆ หรือแต่งขึ้นมาเอง (hallucination)?
- วัดโดย LLM judge (ส่ง context + answer ให้ LLM อีกตัวตรวจ)
- Score 0-1, ยิ่งสูงยิ่งดี — หมายถึง LLM ไม่ hallucinate

**Context Relevance** — เอกสารที่ดึงมาเกี่ยวข้องกับคำถามจริงๆ ไหม?
- วัดคุณภาพของ Retriever
- ถ้า context relevance ต่ำ ต้องปรับ retrieval strategy หรือ chunk strategy

**Answer Relevancy** — คำตอบตอบคำถามที่ถามจริงๆ ไหม (ไม่ใช่ off-topic)?

**MRR (Mean Reciprocal Rank)** — เอกสารที่ถูกต้องอยู่ลำดับแรกๆ ของ retrieval ไหม?

**SAS (Semantic Answer Similarity)** — คำตอบมีความหมายใกล้เคียง ground truth ไหม (เปรียบเทียบ semantic ไม่ใช่ exact match)?

### Haystack Built-in Evaluators

```python
from haystack.components.evaluators import (
    FaithfulnessEvaluator,      # ต้องการ LLM judge
    ContextRelevanceEvaluator,
    AnswerExactMatchEvaluator,
    DocumentMRREvaluator,
    DocumentRecallEvaluator,
    SASEvaluator
)
```

### RAGAS — Open-source Evaluation Framework

RAGAS ให้ metrics ครบในชุดเดียว:
```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_recall

data = {
    "question": questions,
    "answer": responses,
    "contexts": retrieved_contexts,
    "ground_truth": ground_truths
}
result = evaluate(Dataset.from_dict(data), metrics=[faithfulness, answer_relevancy, context_recall])
```

### Langfuse สำหรับ Continuous Evaluation

Langfuse ทำ tracing + evaluation รวมกัน:
- Track query, retrieved docs, prompt, LLM response, latency, cost ทุก request
- Run evaluations แบบ async บน historical data
- ดู trends ของ metrics ตามเวลา

## ประเด็นสำคัญ

- **ต้องมี test dataset** (questions + ground truth answers) ก่อนจะ evaluate ได้
- **LLM judge** ใน Faithfulness/Context Relevance เพิ่ม cost — ใช้ GPT-4o-mini ประหยัดกว่า GPT-4o
- **RAGAS ดีกว่า built-in** สำหรับ comprehensive evaluation — แต่ต้อง install แยก
- **Evaluate เป็นระยะ** หลัง update index หรือเปลี่ยน prompt — ไม่ใช่แค่ครั้งเดียว
- **Retrieval quality มักเป็นคอขวด** — ถ้า faithfulness ต่ำ ให้ตรวจ context relevance ก่อน

## ตัวอย่าง / กรณีศึกษา

**Evaluation Checklist สำหรับ RAG:**
1. สร้าง test set 50-100 คำถาม + ground truth
2. รัน RAG pipeline ทุกข้อ เก็บ (question, retrieved_docs, answer)
3. วัด Context Relevance → ถ้าต่ำ ปรับ retrieval
4. วัด Faithfulness → ถ้าต่ำ ปรับ prompt หรือ top_k
5. วัด Exact Match / SAS → ถ้าต่ำ ปรับ generator parameters

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/rag-retrieval-augmented-generation.md|RAG]] — evaluation เป็นส่วนสำคัญของ RAG lifecycle
- [[wiki/concepts/haystack-framework.md|Haystack Framework]] — built-in evaluators + RAGAS integration

## แหล่งที่มา

- [[wiki/sources/haystack-phase5-production.md|Haystack Phase 5 — Custom Components, Production & Integrations]]
- [[wiki/sources/haystack-phase6-observability.md|Haystack Phase 6 — API, Auth & Observability]]

## จาก RAG คู่มือฉบับสมบูรณ์

### 4 RAGAS Metrics สรุปสั้น

| ตัววัด | วัดอะไร | ถ้าต่ำ → ปรับอะไร |
|--------|--------|-----------------|
| Faithfulness | คำตอบตรงกับ source ไหม? | ปรับ prompt, top_k |
| Answer Relevancy | คำตอบตรงคำถามไหม? | ปรับ system prompt |
| Context Precision | ดึง context ถูกต้องไหม? | ปรับ retrieval strategy |
| Context Recall | ดึง context มาครบไหม? | เพิ่ม top_k, ปรับ chunking |

### Minimal RAGAS Example

```python
from llama_index.core.evaluation import FaithfulnessEvaluator, RelevancyEvaluator

faith_eval = FaithfulnessEvaluator()
result = faith_eval.evaluate_response(response=response)
print(f"Faithful: {result.passing}")  # True/False
```

- [[wiki/sources/rag-complete-guide-comprehensive|RAG คู่มือฉบับสมบูรณ์]]

## จาก Advanced Evaluation Deep Dive — LLM-as-Judge

### เมื่อไหร่ใช้ LLM-as-Judge vs Rule-based

| Rule-based | LLM-as-Judge |
|------------|-------------|
| Exact match (รหัส, ตัวเลข) | คุณภาพการเขียน |
| Format check (JSON valid) | ความเกี่ยวข้อง (relevance) |
| Length check / keyword | ความครบถ้วน (completeness) |
| ถูก/ผิด ชัดเจน | Hallucination detection / Tone |

### Judge Prompt Design ที่ดี

โครงสร้างที่ต้องมี: **Role + Criteria + Scale (rubric) + Input + Output format**

```
# ❌ ไม่ดี
"คำตอบนี้ดีไหม? ให้คะแนน 1-10"

# ✅ ดี
"""เกณฑ์ "ความเกี่ยวข้อง":
  5 = ตอบตรงคำถาม ครบถ้วน
  4 = ตอบตรงแต่ขาดรายละเอียดเล็กน้อย
  ...
SCORE: <1-5>
REASON: <เหตุผล>"""
```

### RAG Triad — 3 มิติหลัก

```python
def rag_triad_judge(run, example):
    # 1. CONTEXT_RELEVANCE: Context ที่ดึงมาเกี่ยวข้องกับคำถามไหม?
    # 2. FAITHFULNESS: คำตอบ AI มาจาก Context จริงไหม (ไม่ hallucinate)?
    # 3. ANSWER_RELEVANCE: คำตอบตอบคำถามได้ถูกต้องไหม?
```

### Multi-Dimension Grading (ประหยัด cost)

ให้คะแนน 4 มิติ (relevance/completeness/clarity/accuracy) ใน **1 LLM call** แทนที่จะ call แยก 4 ครั้ง

### Pitfalls ของ LLM-as-Judge

- **Position Bias**: LLM ให้คะแนน "คำตอบแรก" สูงกว่า → แก้: สลับตำแหน่ง A/B
- **Verbosity Bias**: LLM ให้คะแนน "คำตอบยาว" สูงกว่า → แก้: ระบุใน criteria ว่าความยาวไม่ใช่เกณฑ์
- **Self-preference Bias**: LLM ให้คะแนน output ของ model เดียวกันสูงกว่า → แก้: ใช้ judge model คนละตัว
- **Inconsistency**: รัน 2 ครั้งได้คะแนนต่างกัน → แก้: ใช้ `temperature=0` + รันหลายครั้งแล้วเฉลี่ย

### Agent-specific Evaluators

- **Tool Selection**: exact match — `expected_tool in tool_calls` → score 1.0/0.0
- **Task Completion**: LLM-as-Judge ตรวจว่า agent ทำ task สำเร็จตาม expected outcome ไหม

- [[wiki/sources/rag-advanced-evaluation-deepdive|Advanced Evaluation Deep Dive — LLM-as-Judge]]
