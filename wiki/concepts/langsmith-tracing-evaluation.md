---
title: "LangSmith — Tracing & Evaluation Platform"
type: concept
tags: [langsmith, observability, tracing, evaluation, llm-judge, langchain, ci-cd]
sources: [advanced-evaluation-deep-dive.md, langchain-deep-dive.md, lang-ecosystem-complete.md]
related: [wiki/concepts/langchain-framework.md, wiki/concepts/rag-evaluation.md]
created: 2026-04-14
updated: 2026-04-14
---

## สรุปสั้น

LangSmith คือ platform ของ LangChain สำหรับ Observability, Tracing, และ Evaluation ของ LLM Applications — เปิดใช้ด้วย 2 environment variables, ดู dashboard ที่แสดง input→retrieval→prompt→LLM→output พร้อม latency, tokens, cost, และรัน automated evaluation กับ datasets ได้

## อธิบาย

LangSmith ทำงานร่วมกับ LangChain แบบ automatic tracing — ทุก chain/agent ที่รันจะถูก record โดยอัตโนมัติ เหมาะสำหรับ debug, monitor production, และ run structured evaluations

### เปิดใช้งาน

```python
import os
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "ls_..."
# optional:
os.environ["LANGCHAIN_PROJECT"] = "my-project-name"
```

### Evaluation API

```python
from langsmith import Client, evaluate

client = Client()

# 1. สร้าง Dataset
dataset = client.create_dataset("my-eval-dataset")
client.create_example(
    inputs={"question": "ลาพักร้อนกี่วัน?"},
    outputs={"answer": "15 วันต่อปี"},
    dataset_id=dataset.id
)

# 2. รัน Evaluation
results = evaluate(
    my_bot_function,            # function รับ inputs dict → outputs dict
    data="my-eval-dataset",
    evaluators=[relevance_eval, faithfulness_eval],
    experiment_prefix="v2-test",
    max_concurrency=4,
)
```

## ประเด็นสำคัญ

- **Automatic Tracing**: เพียง set 2 env vars — ทุก LangChain operation ถูก log อัตโนมัติ
- **Dashboard**: เห็น full trace — Input → Retrieval → Prompt → LLM → Output, latency แต่ละ step, token usage, cost
- **Evaluator Contract**: evaluator function รับ `(run, example)` → return `{"key": str, "score": float 0-1, "comment": str}`
- **Multi-evaluator**: ส่ง list ของ evaluators — รันทั้งหมดพร้อมกันสำหรับแต่ละ test case
- **`max_concurrency`**: รัน test cases แบบ parallel — เพิ่มความเร็ว
- **Experiment Prefix**: แยก experiment ตามชื่อ เปรียบเทียบ version ต่างๆ บน dashboard ได้

## ตัวอย่าง / กรณีศึกษา

### LLM-as-Judge Evaluator สำหรับ LangSmith

```python
from langchain_google_genai import ChatGoogleGenerativeAI

judge_model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

def relevance_judge(run, example) -> dict:
    question = example.inputs.get("question", "")
    predicted = run.outputs.get("answer", "")

    prompt = f"""ตรวจ "ความเกี่ยวข้อง":
5 = ตอบตรงคำถาม ครบถ้วน
4 = ตอบตรงแต่ขาดรายละเอียดเล็กน้อย
3 = เกี่ยวข้องบ้าง ไม่ครบ
2 = เกี่ยวข้องน้อย
1 = ไม่ตอบคำถาม

คำถาม: {question}
คำตอบ: {predicted}

SCORE: <1-5>
REASON: <เหตุผล>"""

    response = judge_model.invoke(prompt)
    # parse SCORE: จาก response
    score = parse_score(response.content)
    return {"key": "relevance", "score": score / 5, "comment": "..."}
```

### CI/CD Integration

```python
# eval_pipeline.py — รันใน CI/CD
PASSING_THRESHOLD = 0.7

results = evaluate(my_bot, data="eval-dataset", evaluators=[...])
# คำนวณ avg scores → exit(1) ถ้าต่ำกว่า threshold
```

```bash
python eval_pipeline.py  # exit code 0=pass, 1=fail
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/langchain-framework|LangChain Framework]] — LangSmith เป็นส่วนหนึ่งของ LangChain ecosystem
- [[wiki/concepts/rag-evaluation|RAG Evaluation]] — LangSmith ใช้เป็น platform สำหรับ RAG evaluation pipeline

## แหล่งที่มา

- [[wiki/sources/rag/rag-advanced-evaluation-deepdive|Advanced Evaluation Deep Dive — LLM-as-Judge]]
