# LangSmith Deep Dive — Observability & Evaluation

> LangSmith คือ Platform สำหรับ Tracing, Debugging, Testing และ Monitoring LLM Applications
> ช่วยให้รู้ว่า Agent ทำอะไร ทำไมถึงตอบแบบนั้น และจะปรับปรุงอย่างไร

---

## สารบัญ

1. [LangSmith คืออะไร ทำไมต้องใช้](#1-langsmith-คืออะไร)
2. [Setup & Configuration](#2-setup)
3. [Tracing — ติดตามทุกขั้นตอน](#3-tracing)
4. [Projects & Organization](#4-projects)
5. [Datasets — ชุดข้อมูลทดสอบ](#5-datasets)
6. [Evaluation — วัดคุณภาพอย่างเป็นระบบ](#6-evaluation)
7. [Custom Evaluators](#7-custom-evaluators)
8. [LLM-as-Judge — ใช้ AI ตรวจ AI](#8-llm-as-judge)
9. [Online Evaluation — ตรวจใน Production](#9-online-evaluation)
10. [Monitoring & Alerting](#10-monitoring)
11. [Prompt Hub — จัดการ Prompts](#11-prompt-hub)
12. [Best Practices](#12-best-practices)

---

## 1. LangSmith คืออะไร

### ปัญหาที่ LangSmith แก้

```
❌ ปัญหาทั่วไปเวลาสร้าง LLM App:
  - Agent ตอบผิด แต่ไม่รู้ว่าผิดตรงไหน
  - เปลี่ยน prompt แล้ว ไม่รู้ว่าดีขึ้นหรือแย่ลง
  - ใน production ไม่รู้ว่า user ถามอะไร AI ตอบอะไร
  - เรียก API ช้า แต่ไม่รู้ว่าช้าที่ขั้นตอนไหน
  - ไม่รู้ว่าใช้ token ไปเท่าไหร่

✅ LangSmith แก้ปัญหาเหล่านี้ด้วย:
  - Tracing: ดู trace ทุก step แบบ real-time
  - Evaluation: วัดคุณภาพด้วย dataset + evaluator
  - Monitoring: dashboard สำหรับ production
  - Prompt Management: จัดการ prompt version
```

### สิ่งที่ LangSmith ทำได้

| ความสามารถ | รายละเอียด |
|-----------|----------|
| **Tracing** | ดูทุก LLM call, tool call, chain step พร้อม input/output/latency/tokens |
| **Debugging** | คลิกเข้าไปดูแต่ละ step ว่าส่ง prompt อะไร ได้คำตอบอะไร |
| **Evaluation** | สร้างชุดทดสอบ รัน eval วัดคุณภาพแบบอัตโนมัติ |
| **Comparison** | เปรียบเทียบ 2 versions ว่าอันไหนดีกว่า |
| **Monitoring** | ดู latency, error rate, token usage แบบ real-time |
| **Annotation** | ให้คนมา label/review ผลลัพธ์ของ AI |
| **Prompt Hub** | เก็บ prompt แบบมี version control |

---

## 2. Setup

### 2.1 สมัครและสร้าง API Key

1. ไปที่ [smith.langchain.com](https://smith.langchain.com)
2. สมัครสมาชิก (มี free tier)
3. ไปที่ Settings → API Keys → Create API Key
4. Copy key

### 2.2 ติดตั้ง

```bash
pip install -U langsmith
```

### 2.3 ตั้งค่า Environment Variables

```env
# .env
LANGSMITH_API_KEY=lsv2_pt_xxxxxxxxxxxxxxxx
LANGSMITH_TRACING=true
LANGSMITH_PROJECT=my-first-project
LANGSMITH_ENDPOINT=https://api.smith.langchain.com   # (default, ไม่ต้องใส่ก็ได้)
```

```python
# ใน code
import os
from dotenv import load_dotenv
load_dotenv()

# แค่นี้! LangChain จะส่ง trace ไป LangSmith อัตโนมัติ
```

### 2.4 ตรวจสอบว่าเชื่อมต่อสำเร็จ

```python
from langsmith import Client

client = Client()
print(client.list_projects())  # ถ้าเห็น list → เชื่อมต่อสำเร็จ
```

---

## 3. Tracing

### 3.1 Auto Tracing (ไม่ต้องแก้ code)

เมื่อตั้ง `LANGSMITH_TRACING=true` แล้ว ทุก LangChain operation จะถูก trace อัตโนมัติ:

```python
from langchain_google_genai import ChatGoogleGenerativeAI

model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")

# ทุก invoke จะถูก trace
response = model.invoke("สวัสดีครับ")
# → trace ถูกส่งไป LangSmith Dashboard โดยอัตโนมัติ
```

**สิ่งที่เห็นบน Dashboard:**

```
📦 Run: ChatGoogleGenerativeAI
├── 📥 Input: [HumanMessage("สวัสดีครับ")]
├── 📤 Output: AIMessage("สวัสดีครับ! มีอะไรให้...")
├── ⏱️ Latency: 1.23s
├── 💰 Tokens: input=6, output=42, total=48
├── 📋 Model: gemini-2.5-flash-lite
└── ✅ Status: success
```

### 3.2 @traceable — Trace Custom Functions

```python
from langsmith import traceable

@traceable(name="process_customer_query")
def process_query(question: str, customer_id: str) -> dict:
    """ประมวลผลคำถามลูกค้า"""

    # Step 1: จำแนกประเภท
    classification = model.invoke(f"จำแนกคำถามนี้: {question}")

    # Step 2: ค้นหาข้อมูล
    search_result = search_database(question)

    # Step 3: สร้างคำตอบ
    answer = model.invoke(f"ตอบคำถาม: {question}\nข้อมูล: {search_result}")

    return {
        "answer": answer.content,
        "classification": classification.content,
        "customer_id": customer_id,
    }

result = process_query("ราคาเท่าไหร่", "C-001")
```

**Trace บน Dashboard (nested):**

```
📦 process_customer_query
├── 📦 ChatGoogleGenerativeAI (จำแนกประเภท)
│   ├── ⏱️ 0.8s
│   └── 📤 "billing"
├── 📦 search_database
│   ├── ⏱️ 0.1s
│   └── 📤 "แพ็คเกจ Basic 299 บาท..."
├── 📦 ChatGoogleGenerativeAI (สร้างคำตอบ)
│   ├── ⏱️ 1.1s
│   └── 📤 "เรามีแพ็คเกจ 3 แบบ..."
├── ⏱️ Total: 2.0s
└── ✅ Success
```

### 3.3 Trace ด้วย Context Manager

```python
from langsmith import trace

def my_function(question):
    with trace("my_custom_operation", inputs={"question": question}) as run:
        # ทำงาน
        result = model.invoke(question)

        # เพิ่ม metadata
        run.metadata["custom_key"] = "custom_value"
        run.end(outputs={"answer": result.content})

    return result.content
```

### 3.4 เพิ่ม Metadata และ Tags

```python
@traceable(
    name="query_handler",
    tags=["production", "v2"],        # tags สำหรับ filter
    metadata={"version": "2.0"},      # metadata เพิ่มเติม
)
def handle_query(question: str):
    return model.invoke(question)

# หรือเพิ่มตอน invoke
response = model.invoke(
    "สวัสดี",
    config={
        "tags": ["test"],
        "metadata": {"user_id": "U-001", "session": "S-123"}
    }
)
```

### 3.5 Trace LangGraph

LangGraph จะถูก trace แบบ nested อัตโนมัติ:

```
📦 LangGraph Run
├── 📦 Node: classify
│   └── 📦 ChatGoogleGenerativeAI
├── 📦 Node: agent
│   └── 📦 ChatGoogleGenerativeAI (with tool_calls)
├── 📦 Node: tools
│   └── 📦 search_database
├── 📦 Node: agent (รอบที่ 2)
│   └── 📦 ChatGoogleGenerativeAI
├── ⏱️ Total: 3.5s
└── ✅ 4 nodes executed
```

### 3.6 ปิด Tracing ชั่วคราว

```python
import os

# ปิด tracing
os.environ["LANGSMITH_TRACING"] = "false"

# เปิดใหม่
os.environ["LANGSMITH_TRACING"] = "true"
```

---

## 4. Projects

### 4.1 จัดกลุ่ม Traces ด้วย Project

```env
# แยก project ตาม environment
LANGSMITH_PROJECT=my-app-production    # production
LANGSMITH_PROJECT=my-app-staging       # staging
LANGSMITH_PROJECT=my-app-development   # development
```

### 4.2 สร้าง Project ด้วย Code

```python
from langsmith import Client

client = Client()

# สร้าง project
client.create_project("customer-support-v2", description="Customer Support Bot version 2")

# ดู projects ทั้งหมด
for project in client.list_projects():
    print(f"{project.name}: {project.run_count} runs")
```

### 4.3 เปลี่ยน Project ตอน Runtime

```python
# เปลี่ยน project สำหรับ invoke เฉพาะ
response = model.invoke(
    "สวัสดี",
    config={"metadata": {"project_name": "experiment-1"}}
)

# หรือใช้ environment variable
import os
os.environ["LANGSMITH_PROJECT"] = "experiment-1"
```

---

## 5. Datasets

### 5.1 สร้าง Dataset

```python
from langsmith import Client

client = Client()

# สร้าง dataset
dataset = client.create_dataset(
    dataset_name="customer-support-qa",
    description="ชุดทดสอบ Q&A สำหรับ Customer Support Bot"
)

print(f"Dataset ID: {dataset.id}")
print(f"Name: {dataset.name}")
```

### 5.2 เพิ่ม Examples

```python
# เพิ่มทีละตัว
client.create_example(
    inputs={"question": "ราคาแพ็คเกจเท่าไหร่?"},
    outputs={"answer": "แพ็คเกจ Basic 299 บาท/เดือน, Pro 599 บาท/เดือน"},
    dataset_id=dataset.id,
)

# เพิ่มหลายตัวพร้อมกัน
examples = [
    {
        "inputs": {"question": "ขอคืนเงินได้ไหม?"},
        "outputs": {"answer": "คืนเงินได้ภายใน 30 วัน", "category": "billing"},
    },
    {
        "inputs": {"question": "ลืมรหัสผ่าน"},
        "outputs": {"answer": "กดลืมรหัสผ่านที่หน้า login", "category": "technical"},
    },
    {
        "inputs": {"question": "เปิดทำการกี่โมง?"},
        "outputs": {"answer": "จันทร์-ศุกร์ 9:00-18:00", "category": "general"},
    },
    {
        "inputs": {"question": "สั่งซื้อสินค้ายังไง?"},
        "outputs": {"answer": "สั่งได้ผ่านเว็บไซต์หรือ Line @company", "category": "general"},
    },
    {
        "inputs": {"question": "มีส่วนลดไหม?"},
        "outputs": {"answer": "สมัครรายปีลด 20%", "category": "billing"},
    },
]

client.create_examples(
    inputs=[e["inputs"] for e in examples],
    outputs=[e["outputs"] for e in examples],
    dataset_id=dataset.id,
)
```

### 5.3 โหลด Dataset จาก CSV

```python
import csv

# อ่าน CSV
with open("test_data.csv", "r", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    for row in reader:
        client.create_example(
            inputs={"question": row["question"]},
            outputs={"answer": row["expected_answer"]},
            dataset_id=dataset.id,
        )
```

### 5.4 จัดการ Dataset

```python
# ดู datasets ทั้งหมด
for ds in client.list_datasets():
    print(f"{ds.name}: {ds.example_count} examples")

# ดู examples ใน dataset
for example in client.list_examples(dataset_name="customer-support-qa"):
    print(f"Q: {example.inputs['question']}")
    print(f"A: {example.outputs['answer']}")
    print()

# ลบ example
# client.delete_example(example_id)

# ลบ dataset
# client.delete_dataset(dataset_id=dataset.id)
```

---

## 6. Evaluation

### 6.1 Evaluation คืออะไร ?

Evaluation คือการรันชุดทดสอบ (Dataset) ผ่าน target function แล้วให้ evaluator ให้คะแนน เพื่อวัดว่า AI ทำงานได้ดีแค่ไหน

```
Dataset (Q&A pairs)
    │
    ▼
Target Function (AI Bot)
    │
    ▼
Evaluators (ตรวจคำตอบ)
    │
    ▼
Results (คะแนน + รายงาน)
```

### 6.2 รัน Evaluation พื้นฐาน

```python
from langsmith import evaluate, Client
from langchain_google_genai import ChatGoogleGenerativeAI

model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

# Target function — ฟังก์ชันที่จะทดสอบ
def my_bot(inputs: dict) -> dict:
    """Bot ที่เราจะทดสอบ"""
    question = inputs["question"]
    response = model.invoke(f"ตอบสั้นๆ: {question}")
    return {"answer": response.content}

# Evaluator — ฟังก์ชันที่ให้คะแนน
def contains_keyword(run, example) -> dict:
    """ตรวจสอบว่าคำตอบมีคำหลักจาก expected answer หรือไม่"""
    predicted = run.outputs.get("answer", "").lower()
    expected = example.outputs.get("answer", "").lower()

    # ตัด expected เป็นคำ แล้วดูว่า predicted มีคำไหนบ้าง
    expected_words = set(expected.split())
    predicted_words = set(predicted.split())
    common = expected_words & predicted_words

    score = len(common) / len(expected_words) if expected_words else 0

    return {"key": "keyword_overlap", "score": score}

# รัน Evaluation
results = evaluate(
    my_bot,                              # target function
    data="customer-support-qa",          # dataset name
    evaluators=[contains_keyword],       # evaluators
    experiment_prefix="v1-gemini-flash", # ชื่อ experiment
)

print(f"Results: {results}")
```

**ผลลัพธ์บน Dashboard:**

```
Experiment: v1-gemini-flash-<timestamp>

| # | Question              | Expected          | Predicted           | keyword_overlap |
|---|-----------------------|-------------------|---------------------|-----------------|
| 1 | ราคาแพ็คเกจเท่าไหร่?  | แพ็คเกจ Basic 299... | เรามี 3 แพ็คเกจ... | 0.67            |
| 2 | ขอคืนเงินได้ไหม?      | คืนเงินได้ภายใน 30... | สามารถคืนเงินได้... | 0.75            |
| 3 | ลืมรหัสผ่าน            | กดลืมรหัสผ่าน...     | ไปที่หน้า login...   | 0.50            |
| 4 | เปิดทำการกี่โมง?       | จันทร์-ศุกร์ 9:00... | เปิดให้บริการ...    | 0.60            |
| 5 | มีส่วนลดไหม?          | สมัครรายปีลด 20%   | มีโปรลดราคา...      | 0.40            |

Average keyword_overlap: 0.584
```

---

## 7. Custom Evaluators

### 7.1 โครงสร้างของ Evaluator

```python
def my_evaluator(run, example) -> dict:
    """
    Parameters:
    - run: ผลลัพธ์จาก target function
        - run.outputs: dict ที่ target function return
        - run.inputs: dict input ที่ส่งเข้า
    - example: ตัวอย่างจาก dataset
        - example.inputs: input ของ example
        - example.outputs: expected output

    Returns:
    - dict ที่มี:
        - "key": ชื่อ metric (เช่น "accuracy", "relevance")
        - "score": คะแนน (float 0-1 หรือ bool True/False)
        - "comment": (optional) คำอธิบาย
    """
    return {"key": "my_metric", "score": 0.8, "comment": "ดี"}
```

### 7.2 ตัวอย่าง Evaluators หลากหลาย

```python
# --- Exact Match ---
def exact_match(run, example) -> dict:
    predicted = run.outputs.get("answer", "").strip()
    expected = example.outputs.get("answer", "").strip()
    return {
        "key": "exact_match",
        "score": 1.0 if predicted == expected else 0.0
    }

# --- Contains Expected ---
def contains_expected(run, example) -> dict:
    predicted = run.outputs.get("answer", "").lower()
    expected = example.outputs.get("answer", "").lower()
    return {
        "key": "contains_expected",
        "score": 1.0 if expected in predicted else 0.0
    }

# --- Length Check ---
def reasonable_length(run, example) -> dict:
    answer = run.outputs.get("answer", "")
    word_count = len(answer.split())
    score = 1.0 if 10 <= word_count <= 200 else 0.0
    return {
        "key": "reasonable_length",
        "score": score,
        "comment": f"{word_count} words"
    }

# --- Category Match ---
def category_match(run, example) -> dict:
    predicted_cat = run.outputs.get("category", "")
    expected_cat = example.outputs.get("category", "")
    return {
        "key": "category_accuracy",
        "score": 1.0 if predicted_cat == expected_cat else 0.0
    }

# --- JSON Valid ---
def valid_json(run, example) -> dict:
    import json
    try:
        json.loads(run.outputs.get("answer", ""))
        return {"key": "valid_json", "score": 1.0}
    except:
        return {"key": "valid_json", "score": 0.0}

# รัน evaluators หลายตัวพร้อมกัน
results = evaluate(
    my_bot,
    data="customer-support-qa",
    evaluators=[exact_match, contains_expected, reasonable_length],
    experiment_prefix="multi-eval"
)
```

---

## 8. LLM-as-Judge

### 8.1 ใช้ LLM ตรวจ LLM

เมื่อ rule-based evaluator ไม่เพียงพอ ใช้ LLM เป็นผู้ตัดสิน:

```python
from langchain_google_genai import ChatGoogleGenerativeAI

judge_model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

def llm_relevance_judge(run, example) -> dict:
    """ใช้ LLM ตัดสินว่าคำตอบตรงประเด็นหรือไม่"""
    question = example.inputs["question"]
    predicted = run.outputs.get("answer", "")
    expected = example.outputs.get("answer", "")

    judge_prompt = f"""เป็นผู้ตัดสิน ให้คะแนนความตรงประเด็นของคำตอบ AI (0-10):

คำถาม: {question}
คำตอบที่คาดหวัง: {expected}
คำตอบจาก AI: {predicted}

ให้คะแนน 0-10 และเหตุผลสั้นๆ
ตอบในรูปแบบ:
SCORE: <ตัวเลข>
REASON: <เหตุผล>"""

    response = judge_model.invoke(judge_prompt)
    text = response.content

    # Parse score
    try:
        score_line = [l for l in text.split("\n") if "SCORE:" in l][0]
        score = float(score_line.split("SCORE:")[-1].strip()) / 10  # normalize to 0-1
    except:
        score = 0.5

    # Parse reason
    try:
        reason_line = [l for l in text.split("\n") if "REASON:" in l][0]
        reason = reason_line.split("REASON:")[-1].strip()
    except:
        reason = "ไม่สามารถ parse ได้"

    return {"key": "llm_relevance", "score": score, "comment": reason}
```

### 8.2 ตรวจหลายมิติพร้อมกัน

```python
def llm_multi_judge(run, example) -> list[dict]:
    """ตรวจหลายมิติ: relevance, completeness, tone"""
    question = example.inputs["question"]
    predicted = run.outputs.get("answer", "")

    judge_prompt = f"""ตรวจคำตอบนี้ใน 3 มิติ ให้คะแนนแต่ละมิติ 0-10:

คำถาม: {question}
คำตอบ: {predicted}

ตอบในรูปแบบ:
RELEVANCE: <0-10>
COMPLETENESS: <0-10>
TONE: <0-10>"""

    response = judge_model.invoke(judge_prompt)
    text = response.content

    results = []
    for metric in ["RELEVANCE", "COMPLETENESS", "TONE"]:
        try:
            line = [l for l in text.split("\n") if metric in l][0]
            score = float(line.split(":")[-1].strip()) / 10
        except:
            score = 0.5
        results.append({"key": metric.lower(), "score": score})

    return results
```

---

## 9. Online Evaluation

### 9.1 ตรวจคุณภาพใน Production

```python
from langsmith import Client
from langsmith.run_helpers import traceable

client = Client()

@traceable
def production_bot(question: str) -> str:
    response = model.invoke(question)
    return response.content

# เมื่อ bot ตอบใน production → trace จะถูกส่งไป LangSmith
# สามารถสร้าง Online Evaluator ใน Dashboard เพื่อตรวจทุก trace อัตโนมัติ
```

### 9.2 สร้าง Rules ใน Dashboard

บน LangSmith Dashboard สามารถสร้าง Automation Rules:

```
Rule: "Flag Negative Sentiment"
  Condition: LLM evaluator detects negative user sentiment
  Action: Add tag "needs-review"

Rule: "Alert Long Latency"
  Condition: Total latency > 10 seconds
  Action: Send webhook to Slack

Rule: "Flag Hallucination"
  Condition: LLM judge scores relevance < 0.3
  Action: Add to "review-queue" dataset
```

---

## 10. Monitoring

### 10.1 Dashboard Metrics

LangSmith Dashboard แสดง:

```
📊 Overview (เลือกช่วงเวลาได้)
├── Total Runs: 15,234
├── Success Rate: 98.5%
├── Avg Latency: 2.1s
├── P99 Latency: 8.3s
├── Total Tokens: 2.3M
├── Estimated Cost: $12.50
├── Error Rate: 1.5%
└── Most Common Errors: Rate limit (60%), Timeout (25%), Other (15%)
```

### 10.2 Filter & Search Traces

```python
# ค้นหา traces ด้วย code
runs = client.list_runs(
    project_name="my-app-production",
    filter='and(gt(latency, 5), eq(status, "error"))',  # latency > 5s AND error
    start_time=datetime(2025, 3, 1),
    end_time=datetime(2025, 3, 31),
    limit=100,
)

for run in runs:
    print(f"Run {run.id}: latency={run.latency}s, error={run.error}")
```

### 10.3 Export Data

```python
# Export traces เป็น DataFrame สำหรับวิเคราะห์
import pandas as pd

runs = list(client.list_runs(
    project_name="my-app-production",
    limit=1000,
))

df = pd.DataFrame([{
    "id": r.id,
    "latency": r.total_tokens,
    "tokens": r.total_tokens,
    "status": r.status,
    "created": r.start_time,
} for r in runs])

print(df.describe())
df.to_csv("traces_export.csv")
```

---

## 11. Prompt Hub

### 11.1 Push Prompt ไป Hub

```python
from langchain import hub
from langchain_core.prompts import ChatPromptTemplate

# สร้าง prompt
prompt = ChatPromptTemplate.from_messages([
    ("system", "คุณคือ Customer Support Bot ของบริษัท {company} ตอบสุภาพ"),
    ("human", "{question}"),
])

# Push ไป Hub (ต้อง login LangSmith ก่อน)
hub.push("my-org/customer-support-prompt", prompt, new_repo_is_public=False)
```

### 11.2 Pull Prompt จาก Hub

```python
# Pull มาใช้ (ทุก environment ใช้ prompt เดียวกัน)
prompt = hub.pull("my-org/customer-support-prompt")

chain = prompt | model
response = chain.invoke({"company": "ABC Corp", "question": "ราคาเท่าไหร่?"})
```

### 11.3 Version Control

```python
# Pull version เฉพาะ
prompt_v1 = hub.pull("my-org/customer-support-prompt:v1")
prompt_v2 = hub.pull("my-org/customer-support-prompt:v2")
prompt_latest = hub.pull("my-org/customer-support-prompt:latest")
```

---

## 12. Best Practices

### 12.1 Tracing Best Practices

```python
# ✅ ตั้ง project name ที่สื่อความหมาย
os.environ["LANGSMITH_PROJECT"] = "support-bot-prod"  # ไม่ใช่ "default"

# ✅ ใส่ metadata ที่มีประโยชน์
@traceable(metadata={"version": "2.1", "team": "ai"})
def my_bot(q):
    return model.invoke(q)

# ✅ ใส่ tags สำหรับ filter
response = model.invoke("...", config={"tags": ["production", "thai-language"]})

# ✅ ปิด tracing เวลาทดสอบ local (ลด noise)
os.environ["LANGSMITH_TRACING"] = "false"  # local dev
os.environ["LANGSMITH_TRACING"] = "true"   # staging/prod
```

### 12.2 Evaluation Best Practices

```python
# ✅ สร้าง Dataset ที่ครอบคลุม
# - Happy path (คำถามปกติ)
# - Edge cases (คำถามแปลกๆ)
# - Adversarial (พยายามหลอก)
# - Multi-language (ถ้า support หลายภาษา)

# ✅ ใช้ Evaluator หลายตัว
results = evaluate(
    my_bot,
    data="qa-test",
    evaluators=[
        exact_match,           # rule-based
        contains_expected,     # rule-based
        reasonable_length,     # rule-based
        llm_relevance_judge,   # LLM-as-judge
    ],
)

# ✅ เปรียบเทียบ experiments
# เปลี่ยน model/prompt แล้ว eval ใหม่
# ดูบน Dashboard เพื่อเปรียบเทียบ

# ✅ ทำ eval เป็น CI/CD
# รัน eval ทุกครั้งที่ merge PR
# ถ้าคะแนนต่ำกว่า threshold → fail build
```

### 12.3 Monitoring Best Practices

```
✅ ตั้ง Alert สำหรับ:
  - Error rate > 5%
  - P99 latency > 10s
  - Token usage spike (อาจมี infinite loop)
  - Success rate drop

✅ Review traces สม่ำเสมอ:
  - ดู traces ที่ user ให้ thumbs down
  - ดู traces ที่มี latency สูง
  - ดู traces ที่มี error

✅ สร้าง Dataset จาก production:
  - นำ traces ที่น่าสนใจมาเพิ่มใน dataset
  - ใช้ human annotation ช่วยให้คะแนน
```

---

## Quick Reference

| ต้องการ | ใช้ |
|---------|-----|
| เปิด auto tracing | `LANGSMITH_TRACING=true` |
| Trace custom function | `@traceable` |
| สร้าง dataset | `client.create_dataset(...)` |
| เพิ่ม test case | `client.create_example(...)` |
| รัน evaluation | `evaluate(target, data, evaluators)` |
| ใช้ LLM ตรวจ | สร้าง evaluator ที่เรียก LLM ภายใน |
| จัดการ prompt | `hub.push(...)` / `hub.pull(...)` |
| ค้นหา traces | `client.list_runs(filter=...)` |
| ดู state history | Dashboard → Traces → click run |
