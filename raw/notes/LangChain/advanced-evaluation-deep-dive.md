# Advanced Evaluation Deep Dive
## สร้าง Custom Evaluator ที่ใช้ LLM เป็น Judge (LLM-as-Judge)

> เมื่อ rule-based evaluator ไม่เพียงพอ ใช้ LLM ตัดสินคุณภาพของ AI อีกที
> เทคนิคนี้เป็นมาตรฐานใน Production LLM Application

---

## สารบัญ

1. [ทำไมต้อง LLM-as-Judge](#1-ทำไมต้อง-llm-as-judge)
2. [หลักการออกแบบ Judge Prompt](#2-judge-prompt-design)
3. [Single-Point Grading — ให้คะแนนทีละข้อ](#3-single-point-grading)
4. [Multi-Dimension Grading — ให้คะแนนหลายมิติ](#4-multi-dimension-grading)
5. [Pairwise Comparison — เปรียบเทียบ 2 ตัว](#5-pairwise-comparison)
6. [Reference-based Judge — เทียบกับคำตอบที่ถูก](#6-reference-based)
7. [RAG-specific Evaluators](#7-rag-evaluators)
8. [Agent-specific Evaluators](#8-agent-evaluators)
9. [รวมกับ LangSmith — End-to-End Evaluation](#9-langsmith-integration)
10. [Evaluation Pipeline แบบอัตโนมัติ](#10-pipeline)
11. [ข้อจำกัดและ Pitfalls](#11-pitfalls)
12. [Best Practices](#12-best-practices)

---

## 1. ทำไมต้อง LLM-as-Judge

### ข้อจำกัดของ Rule-based Evaluation

```
Rule-based:
  ❌ "สวัสดีครับ" ≠ "สวัสดีค่ะ"     → exact match บอกว่าผิด (แต่ถูก)
  ❌ "15 วัน" ≠ "สิบห้าวัน"          → keyword match พลาด
  ❌ ตรวจ "คุณภาพการเขียน" ไม่ได้
  ❌ ตรวจ "ความสุภาพ" ไม่ได้
  ❌ ตรวจ "ตอบตรงคำถามไหม" ยาก

LLM-as-Judge:
  ✅ เข้าใจความหมาย (semantic understanding)
  ✅ ตรวจได้หลายมิติ (relevance, tone, completeness)
  ✅ ให้เหตุผลได้ (explainable)
  ✅ ยืดหยุ่น (ปรับ criteria ได้)
  ⚠️ มี bias (ต้องระวัง)
  ⚠️ มีค่าใช้จ่าย (ต้อง call LLM)
```

### เมื่อไหร่ใช้อะไร

```
Rule-based เหมาะกับ:              LLM-as-Judge เหมาะกับ:
- Exact match (รหัส, ตัวเลข)      - คุณภาพการเขียน
- Format check (JSON valid)       - ความเกี่ยวข้อง (relevance)
- Length check                    - ความครบถ้วน (completeness)
- Contains keyword                - Hallucination detection
- ถูก/ผิด ชัดเจน                  - Tone / ความสุภาพ
                                  - เปรียบเทียบ 2 versions
```

---

## 2. Judge Prompt Design

### 2.1 โครงสร้าง Judge Prompt ที่ดี

```
1. Role:        บอก LLM ว่าเป็นผู้ตัดสิน
2. Criteria:    กำหนดเกณฑ์การให้คะแนนชัดเจน
3. Scale:       กำหนดช่วงคะแนน + คำอธิบายแต่ละระดับ
4. Input:       ให้ดูคำถาม + คำตอบ (+ reference ถ้ามี)
5. Output:      กำหนด format การตอบ
```

### 2.2 ตัวอย่าง Judge Prompt ที่ดี vs ไม่ดี

```python
# ❌ ไม่ดี — กว้างเกินไป ไม่มีเกณฑ์ชัดเจน
bad_prompt = "คำตอบนี้ดีไหม? ให้คะแนน 1-10"

# ✅ ดี — เกณฑ์ชัดเจน มี rubric
good_prompt = """คุณคือผู้ตัดสินคุณภาพคำตอบ AI

เกณฑ์การให้คะแนน "ความเกี่ยวข้อง" (Relevance):
  5 = ตอบตรงคำถาม ครบถ้วน ไม่มีข้อมูลเกิน
  4 = ตอบตรงคำถาม แต่อาจขาดรายละเอียดเล็กน้อย
  3 = ตอบเกี่ยวข้องบ้าง แต่ไม่ครบ
  2 = ตอบเกี่ยวข้องน้อย หรือมีข้อมูลผิดปน
  1 = ไม่ตอบคำถาม หรือตอบผิดทั้งหมด

คำถาม: {question}
คำตอบ: {answer}

ตอบในรูปแบบ:
SCORE: <1-5>
REASON: <เหตุผลสั้นๆ 1-2 ประโยค>"""
```

---

## 3. Single-Point Grading

### 3.1 Evaluator พื้นฐาน

```python
import os
from dotenv import load_dotenv
from langchain_google_genai import ChatGoogleGenerativeAI
from langsmith import Client, evaluate

load_dotenv()

# ใช้ model ที่แม่นยำเป็น judge (อาจใช้คนละตัวกับ target)
judge_model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

def relevance_judge(run, example) -> dict:
    """ตรวจความเกี่ยวข้องของคำตอบ"""
    question = example.inputs.get("question", "")
    predicted = run.outputs.get("answer", "")

    prompt = f"""คุณคือผู้ตัดสินคุณภาพคำตอบ AI

เกณฑ์ "ความเกี่ยวข้อง" (Relevance):
  5 = ตอบตรงคำถาม ครบถ้วน ไม่มีข้อมูลเกิน
  4 = ตอบตรงคำถาม แต่อาจขาดรายละเอียดเล็กน้อย
  3 = ตอบเกี่ยวข้องบ้าง แต่ไม่ครบ
  2 = ตอบเกี่ยวข้องน้อย หรือมีข้อมูลผิดปน
  1 = ไม่ตอบคำถาม หรือตอบผิดทั้งหมด

คำถาม: {question}
คำตอบจาก AI: {predicted}

ตอบในรูปแบบนี้เท่านั้น:
SCORE: <1-5>
REASON: <เหตุผล>"""

    response = judge_model.invoke(prompt)
    text = response.content

    # Parse
    score = 3  # default
    reason = ""
    for line in text.strip().split("\n"):
        if line.startswith("SCORE:"):
            try:
                score = int(line.split(":")[-1].strip())
            except:
                pass
        elif line.startswith("REASON:"):
            reason = line.split(":", 1)[-1].strip()

    return {
        "key": "relevance",
        "score": score / 5,  # normalize to 0-1 สำหรับ LangSmith
        "comment": f"Score {score}/5: {reason}",
    }
```

### 3.2 Helper Function สำหรับ Parse

```python
def parse_judge_response(text: str, max_score: int = 5) -> tuple[int, str]:
    """Parse SCORE: และ REASON: จาก judge response"""
    score = max_score // 2  # default กลางๆ
    reason = "Parse failed"

    for line in text.strip().split("\n"):
        line = line.strip()
        if line.upper().startswith("SCORE:"):
            try:
                s = line.split(":")[-1].strip()
                # handle "4/5" format
                if "/" in s:
                    s = s.split("/")[0]
                score = min(max(int(float(s)), 1), max_score)
            except:
                pass
        elif line.upper().startswith("REASON:"):
            reason = line.split(":", 1)[-1].strip()

    return score, reason

def create_judge_evaluator(
    name: str,
    criteria_prompt: str,
    max_score: int = 5,
    judge_model=None,
):
    """Factory function สร้าง judge evaluator"""
    if judge_model is None:
        judge_model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

    def evaluator(run, example) -> dict:
        question = example.inputs.get("question", "")
        predicted = run.outputs.get("answer", "")
        expected = example.outputs.get("answer", "")

        prompt = criteria_prompt.format(
            question=question,
            predicted=predicted,
            expected=expected,
        )
        response = judge_model.invoke(prompt)
        score, reason = parse_judge_response(response.content, max_score)

        return {
            "key": name,
            "score": score / max_score,  # normalize 0-1
            "comment": f"{score}/{max_score}: {reason}",
        }

    return evaluator
```

### 3.3 สร้าง Evaluators ด้วย Factory

```python
# === Relevance ===
relevance_eval = create_judge_evaluator(
    name="relevance",
    criteria_prompt="""ตรวจ "ความเกี่ยวข้อง":
5 = ตอบตรงคำถาม ครบถ้วน
4 = ตอบตรงแต่ขาดรายละเอียดเล็กน้อย
3 = เกี่ยวข้องบ้าง ไม่ครบ
2 = เกี่ยวข้องน้อย
1 = ไม่ตอบคำถาม

คำถาม: {question}
คำตอบ: {predicted}

SCORE: <1-5>
REASON: <เหตุผล>""",
)

# === Helpfulness ===
helpfulness_eval = create_judge_evaluator(
    name="helpfulness",
    criteria_prompt="""ตรวจ "ความเป็นประโยชน์":
5 = มีประโยชน์มาก นำไปใช้ได้ทันที
4 = มีประโยชน์ดี แต่อาจต้องหาข้อมูลเพิ่มเล็กน้อย
3 = มีประโยชน์ปานกลาง
2 = มีประโยชน์น้อย
1 = ไม่มีประโยชน์

คำถาม: {question}
คำตอบ: {predicted}

SCORE: <1-5>
REASON: <เหตุผล>""",
)

# === Tone/Politeness ===
tone_eval = create_judge_evaluator(
    name="tone",
    criteria_prompt="""ตรวจ "ความสุภาพและเป็นมิตร":
5 = สุภาพมาก เป็นมิตร อบอุ่น
4 = สุภาพดี
3 = เฉยๆ ไม่สุภาพไม่หยาบ
2 = ค่อนข้างแข็ง/เย็นชา
1 = หยาบ ไม่สุภาพ

คำตอบ: {predicted}

SCORE: <1-5>
REASON: <เหตุผล>""",
)
```

---

## 4. Multi-Dimension Grading

### 4.1 ให้คะแนนหลายมิติในครั้งเดียว

```python
def multi_dimension_judge(run, example) -> list[dict]:
    """ให้คะแนน 4 มิติพร้อมกัน ใน 1 LLM call (ประหยัดค่าใช้จ่าย)"""

    question = example.inputs.get("question", "")
    predicted = run.outputs.get("answer", "")

    prompt = f"""คุณคือผู้ตัดสินคุณภาพคำตอบ AI ให้คะแนน 4 มิติ (1-5 แต่ละมิติ):

1. RELEVANCE (ความเกี่ยวข้อง): ตอบตรงคำถามไหม
2. COMPLETENESS (ความครบถ้วน): ข้อมูลครบไหม
3. CLARITY (ความชัดเจน): อ่านเข้าใจง่ายไหม
4. ACCURACY (ความถูกต้อง): ข้อมูลถูกต้องไหม

คำถาม: {question}
คำตอบ: {predicted}

ตอบในรูปแบบนี้:
RELEVANCE: <1-5>
COMPLETENESS: <1-5>
CLARITY: <1-5>
ACCURACY: <1-5>
OVERALL_REASON: <เหตุผลรวม 1-2 ประโยค>"""

    response = judge_model.invoke(prompt)
    text = response.content

    results = []
    reason = ""
    for line in text.strip().split("\n"):
        line = line.strip()
        for metric in ["RELEVANCE", "COMPLETENESS", "CLARITY", "ACCURACY"]:
            if line.upper().startswith(metric + ":"):
                try:
                    score = int(line.split(":")[-1].strip().split("/")[0])
                    score = min(max(score, 1), 5)
                except:
                    score = 3
                results.append({
                    "key": metric.lower(),
                    "score": score / 5,
                })
        if line.upper().startswith("OVERALL_REASON:"):
            reason = line.split(":", 1)[-1].strip()

    # เพิ่ม overall score (ค่าเฉลี่ย)
    if results:
        avg = sum(r["score"] for r in results) / len(results)
        results.append({
            "key": "overall",
            "score": avg,
            "comment": reason,
        })

    return results
```

**Output ตัวอย่าง (บน LangSmith Dashboard):**
```
| Question           | relevance | completeness | clarity | accuracy | overall |
|--------------------|-----------|-------------|---------|----------|---------|
| ลาพักร้อนกี่วัน    | 1.0       | 0.8         | 1.0     | 1.0      | 0.95    |
| VPN server คือ     | 1.0       | 1.0         | 0.8     | 1.0      | 0.95    |
| สวัสดิการทั้งหมด    | 0.8       | 0.6         | 0.8     | 0.8      | 0.75    |
```

---

## 5. Pairwise Comparison

### 5.1 เปรียบเทียบ 2 คำตอบ

ใช้เมื่อต้องการเปรียบเทียบ 2 versions (เช่น model A vs model B):

```python
def pairwise_judge(
    question: str,
    answer_a: str,
    answer_b: str,
    judge_model=None,
) -> dict:
    """เปรียบเทียบ 2 คำตอบ"""
    if judge_model is None:
        judge_model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

    # สำคัญ: สลับตำแหน่ง A/B เพื่อลด position bias
    import random
    if random.random() > 0.5:
        first, second = answer_a, answer_b
        first_label, second_label = "A", "B"
    else:
        first, second = answer_b, answer_a
        first_label, second_label = "B", "A"

    prompt = f"""เปรียบเทียบคำตอบ 2 ข้อ สำหรับคำถาม:
"{question}"

--- คำตอบ 1 ---
{first}

--- คำตอบ 2 ---
{second}

เกณฑ์: ความเกี่ยวข้อง ความครบถ้วน ความชัดเจน ความถูกต้อง

ตอบในรูปแบบ:
WINNER: <1 หรือ 2 หรือ TIE>
REASON: <เหตุผล>"""

    response = judge_model.invoke(prompt)
    text = response.content

    winner = "TIE"
    reason = ""
    for line in text.strip().split("\n"):
        if line.upper().startswith("WINNER:"):
            w = line.split(":")[-1].strip()
            if "1" in w:
                winner = first_label  # map กลับเป็น A/B
            elif "2" in w:
                winner = second_label
            else:
                winner = "TIE"
        elif line.upper().startswith("REASON:"):
            reason = line.split(":", 1)[-1].strip()

    return {"winner": winner, "reason": reason}

# ใช้งาน
result = pairwise_judge(
    question="LangChain คืออะไร",
    answer_a="LangChain เป็น Framework สำหรับสร้าง AI Application ที่ทำงานร่วมกับ LLM",
    answer_b="LangChain คือ library สำหรับ AI",
)
print(f"Winner: {result['winner']}")   # "A"
print(f"Reason: {result['reason']}")   # "คำตอบ A ละเอียดกว่า มีบริบทมากกว่า"
```

### 5.2 Pairwise Evaluation กับ LangSmith

```python
from langsmith import evaluate

model_a = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")
model_b = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0.7)

def bot_a(inputs):
    return {"answer": model_a.invoke(inputs["question"]).content}

def bot_b(inputs):
    return {"answer": model_b.invoke(inputs["question"]).content}

# Run evaluation สำหรับทั้ง 2 models
results_a = evaluate(bot_a, data="qa-test", evaluators=[relevance_eval, helpfulness_eval],
                     experiment_prefix="model-A")
results_b = evaluate(bot_b, data="qa-test", evaluators=[relevance_eval, helpfulness_eval],
                     experiment_prefix="model-B")

# เปรียบเทียบบน LangSmith Dashboard:
# model-A: avg relevance 0.85, avg helpfulness 0.80
# model-B: avg relevance 0.78, avg helpfulness 0.82
```

---

## 6. Reference-based Judge

### 6.1 เปรียบเทียบกับคำตอบที่ถูกต้อง

```python
def reference_judge(run, example) -> dict:
    """เปรียบเทียบคำตอบกับ reference answer"""
    question = example.inputs.get("question", "")
    predicted = run.outputs.get("answer", "")
    reference = example.outputs.get("answer", "")

    prompt = f"""เปรียบเทียบ "คำตอบจาก AI" กับ "คำตอบที่ถูกต้อง":

คำถาม: {question}
คำตอบที่ถูกต้อง: {reference}
คำตอบจาก AI: {predicted}

ให้คะแนน 1-5:
5 = ตรงกับคำตอบที่ถูกต้อง สื่อความหมายเดียวกัน แม้ใช้คำต่างกัน
4 = ตรงเป็นส่วนใหญ่ ขาดรายละเอียดเล็กน้อย
3 = ตรงบางส่วน ขาดข้อมูลสำคัญ
2 = ตรงน้อย มีข้อมูลผิดปน
1 = ไม่ตรงเลย หรือตอบผิดทั้งหมด

SCORE: <1-5>
REASON: <เหตุผล>"""

    response = judge_model.invoke(prompt)
    score, reason = parse_judge_response(response.content)

    return {
        "key": "reference_match",
        "score": score / 5,
        "comment": f"{score}/5: {reason}",
    }
```

---

## 7. RAG-specific Evaluators

### 7.1 Faithfulness — ตอบจาก Context จริงไหม

```python
def faithfulness_judge(run, example) -> dict:
    """ตรวจว่าคำตอบมาจาก context ที่ดึงมาจริงไหม (ไม่ hallucinate)"""
    question = example.inputs.get("question", "")
    predicted = run.outputs.get("answer", "")
    context = run.outputs.get("context", "")  # context ที่ดึงมา

    if not context:
        return {"key": "faithfulness", "score": 0.5, "comment": "No context provided"}

    prompt = f"""ตรวจสอบว่าคำตอบ AI มาจาก Context ที่ให้มาหรือไม่:

Context (ข้อมูลที่ดึงมาจากเอกสาร):
{context[:1000]}

คำตอบจาก AI:
{predicted}

ตรวจทีละประโยคในคำตอบ:
- ทุกข้อมูลใน "คำตอบ" สามารถหาที่มาจาก "Context" ได้หรือไม่?
- มีข้อมูลที่ AI แต่งเพิ่มเองไหม?

SCORE:
  5 = ทุกข้อมูลมาจาก Context (Faithful)
  4 = เกือบทั้งหมดมาจาก Context มีการสรุปเล็กน้อย
  3 = ครึ่งหนึ่งมาจาก Context ครึ่งหนึ่งน่าจะแต่ง
  2 = ส่วนใหญ่แต่งเพิ่ม
  1 = แทบไม่มีข้อมูลจาก Context (Hallucination)

SCORE: <1-5>
REASON: <ระบุส่วนที่ hallucinate ถ้ามี>"""

    response = judge_model.invoke(prompt)
    score, reason = parse_judge_response(response.content)

    return {
        "key": "faithfulness",
        "score": score / 5,
        "comment": f"{score}/5: {reason}",
    }
```

### 7.2 Context Relevance — ดึง Context ที่เกี่ยวข้องมาไหม

```python
def context_relevance_judge(run, example) -> dict:
    """ตรวจว่า context ที่ดึงมาเกี่ยวข้องกับคำถามไหม"""
    question = example.inputs.get("question", "")
    context = run.outputs.get("context", "")

    prompt = f"""ตรวจว่าข้อมูลที่ดึงมาเกี่ยวข้องกับคำถามไหม:

คำถาม: {question}

ข้อมูลที่ดึงมา:
{context[:1000]}

SCORE:
  5 = ข้อมูลเกี่ยวข้องทั้งหมด ตรงประเด็น
  4 = เกี่ยวข้องเป็นส่วนใหญ่ มีบางส่วนไม่เกี่ยว
  3 = เกี่ยวข้องบ้าง ไม่เกี่ยวบ้าง
  2 = เกี่ยวข้องน้อย
  1 = ไม่เกี่ยวข้องเลย

SCORE: <1-5>
REASON: <เหตุผล>"""

    response = judge_model.invoke(prompt)
    score, reason = parse_judge_response(response.content)

    return {
        "key": "context_relevance",
        "score": score / 5,
        "comment": f"{score}/5: {reason}",
    }
```

### 7.3 Answer Correctness — ตอบถูกไหม (Semantic)

```python
def semantic_correctness_judge(run, example) -> dict:
    """ตรวจความถูกต้องเชิงความหมาย (ไม่ใช่ exact match)"""
    question = example.inputs.get("question", "")
    predicted = run.outputs.get("answer", "")
    expected = example.outputs.get("answer", "")

    prompt = f"""เปรียบเทียบ "คำตอบจาก AI" กับ "คำตอบที่ถูกต้อง"
โดยพิจารณา "ความหมาย" ไม่ใช่ "คำต่อคำ":

คำถาม: {question}
คำตอบที่ถูกต้อง: {expected}
คำตอบจาก AI: {predicted}

ตัวอย่าง:
- "15 วัน" กับ "สิบห้าวัน" = เหมือนกัน (SCORE: 5)
- "กรุงเทพ" กับ "กรุงเทพมหานคร" = เหมือนกัน (SCORE: 5)
- "ลาได้ 15 วัน" กับ "ลาได้ 10 วัน" = ผิด (SCORE: 1)

SCORE: <1-5>
REASON: <เหตุผล>"""

    response = judge_model.invoke(prompt)
    score, reason = parse_judge_response(response.content)

    return {
        "key": "semantic_correctness",
        "score": score / 5,
        "comment": f"{score}/5: {reason}",
    }
```

### 7.4 RAG Triad — รวม 3 มิติหลักของ RAG

```python
def rag_triad_judge(run, example) -> list[dict]:
    """RAG Triad: Context Relevance + Faithfulness + Answer Relevance"""
    question = example.inputs.get("question", "")
    predicted = run.outputs.get("answer", "")
    context = run.outputs.get("context", "")
    expected = example.outputs.get("answer", "")

    prompt = f"""ตรวจคุณภาพ RAG ใน 3 มิติ:

คำถาม: {question}
Context ที่ดึงมา: {context[:800]}
คำตอบจาก AI: {predicted}
คำตอบที่ถูกต้อง: {expected}

ให้คะแนนแต่ละมิติ 1-5:

1. CONTEXT_RELEVANCE: Context ที่ดึงมาเกี่ยวข้องกับคำถามไหม?
2. FAITHFULNESS: คำตอบ AI มาจาก Context จริงไหม (ไม่ hallucinate)?
3. ANSWER_RELEVANCE: คำตอบตอบคำถามได้ถูกต้องไหม?

CONTEXT_RELEVANCE: <1-5>
FAITHFULNESS: <1-5>
ANSWER_RELEVANCE: <1-5>
REASON: <เหตุผลรวม>"""

    response = judge_model.invoke(prompt)
    text = response.content

    results = []
    reason = ""
    for line in text.strip().split("\n"):
        line = line.strip()
        for metric in ["CONTEXT_RELEVANCE", "FAITHFULNESS", "ANSWER_RELEVANCE"]:
            if line.upper().startswith(metric + ":"):
                try:
                    score = int(line.split(":")[-1].strip().split("/")[0])
                    score = min(max(score, 1), 5)
                except:
                    score = 3
                results.append({"key": metric.lower(), "score": score / 5})
        if line.upper().startswith("REASON:"):
            reason = line.split(":", 1)[-1].strip()

    if results:
        avg = sum(r["score"] for r in results) / len(results)
        results.append({"key": "rag_quality", "score": avg, "comment": reason})

    return results
```

---

## 8. Agent-specific Evaluators

### 8.1 Tool Selection — เลือก Tool ถูกไหม

```python
def tool_selection_judge(run, example) -> dict:
    """ตรวจว่า Agent เลือกใช้ Tool ที่ถูกต้องไหม"""
    question = example.inputs.get("question", "")
    expected_tool = example.outputs.get("expected_tool", "")

    # ดึง tool calls จาก trace
    tool_calls = []
    for msg in run.outputs.get("messages", []):
        if hasattr(msg, "tool_calls") and msg.tool_calls:
            tool_calls.extend([tc["name"] for tc in msg.tool_calls])

    if not expected_tool:
        return {"key": "tool_selection", "score": 1.0, "comment": "No expected tool specified"}

    correct = expected_tool in tool_calls
    return {
        "key": "tool_selection",
        "score": 1.0 if correct else 0.0,
        "comment": f"Expected: {expected_tool}, Used: {tool_calls}",
    }
```

### 8.2 Task Completion — ทำงานสำเร็จไหม

```python
def task_completion_judge(run, example) -> dict:
    """ตรวจว่า Agent ทำ task สำเร็จไหม"""
    question = example.inputs.get("question", "")
    predicted = run.outputs.get("answer", "")
    expected_outcome = example.outputs.get("expected_outcome", "")

    prompt = f"""ตรวจว่า AI Agent ทำงานสำเร็จตามที่คาดหวังไหม:

Task: {question}
Expected Outcome: {expected_outcome}
Actual Result: {predicted}

SCORE:
  5 = สำเร็จทั้งหมด ผลลัพธ์ถูกต้อง
  4 = สำเร็จเป็นส่วนใหญ่ ขาดรายละเอียดเล็กน้อย
  3 = สำเร็จบางส่วน
  2 = สำเร็จน้อย
  1 = ไม่สำเร็จ

SCORE: <1-5>
REASON: <เหตุผล>"""

    response = judge_model.invoke(prompt)
    score, reason = parse_judge_response(response.content)

    return {
        "key": "task_completion",
        "score": score / 5,
        "comment": f"{score}/5: {reason}",
    }
```

---

## 9. LangSmith Integration

### 9.1 End-to-End Evaluation Pipeline

```python
from langsmith import Client, evaluate
from langchain_google_genai import ChatGoogleGenerativeAI

client = Client()

# ======== 1. สร้าง Dataset ========
dataset = client.create_dataset("comprehensive-eval")

test_cases = [
    {
        "inputs": {"question": "ลาพักร้อนได้กี่วัน?"},
        "outputs": {
            "answer": "15 วันต่อปี",
            "expected_tool": "search_hr_docs",
            "expected_outcome": "ตอบจำนวนวันลาพักร้อนได้ถูกต้อง",
        }
    },
    {
        "inputs": {"question": "VPN server address คืออะไร?"},
        "outputs": {
            "answer": "vpn.abc-company.com",
            "expected_tool": "search_it_docs",
            "expected_outcome": "ตอบ VPN server address ได้ถูกต้อง",
        }
    },
    {
        "inputs": {"question": "ค่าอาหาร + ค่าเดินทาง ต่อเดือนเท่าไหร่?"},
        "outputs": {
            "answer": "ค่าอาหาร 2,200 + ค่าเดินทาง 2,500 = 4,700 บาท",
            "expected_tool": "search_hr_docs",
            "expected_outcome": "คำนวณค่าอาหารและค่าเดินทางรวมได้ถูกต้อง",
        }
    },
    {
        "inputs": {"question": "วิธีทำส้มตำ?"},
        "outputs": {
            "answer": "ไม่มีข้อมูล",
            "expected_tool": "",
            "expected_outcome": "ปฏิเสธอย่างสุภาพ บอกว่าไม่มีข้อมูล",
        }
    },
]

for tc in test_cases:
    client.create_example(inputs=tc["inputs"], outputs=tc["outputs"], dataset_id=dataset.id)

# ======== 2. Target Function ========
model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

def my_rag_bot(inputs: dict) -> dict:
    """Bot ที่จะทดสอบ"""
    question = inputs["question"]
    # สมมติว่ามี RAG pipeline
    response = model.invoke(f"ตอบสั้นๆ: {question}")
    return {
        "answer": response.content,
        "context": "mock context...",
    }

# ======== 3. รัน Evaluation ========
results = evaluate(
    my_rag_bot,
    data="comprehensive-eval",
    evaluators=[
        # Rule-based
        lambda run, example: {
            "key": "has_answer",
            "score": 1.0 if len(run.outputs.get("answer", "")) > 10 else 0.0,
        },
        # LLM-as-Judge
        relevance_eval,
        helpfulness_eval,
        semantic_correctness_judge,
        task_completion_judge,
        # Multi-dimension
        multi_dimension_judge,
    ],
    experiment_prefix="rag-bot-v2-comprehensive",
    max_concurrency=4,
)

print("✅ Evaluation complete! Check LangSmith Dashboard.")
```

**Dashboard Result:**
```
Experiment: rag-bot-v2-comprehensive-20250401

| # | Question               | has_answer | relevance | helpfulness | semantic | task     | overall |
|---|------------------------|-----------|-----------|-------------|----------|----------|---------|
| 1 | ลาพักร้อนกี่วัน        | ✅ 1.0    | 0.9       | 0.9         | 0.8      | 0.9      | 0.88    |
| 2 | VPN server             | ✅ 1.0    | 1.0       | 1.0         | 1.0      | 1.0      | 1.00    |
| 3 | ค่าอาหาร+ค่าเดินทาง   | ✅ 1.0    | 0.8       | 0.8         | 0.6      | 0.7      | 0.73    |
| 4 | ส้มตำ (out of scope)   | ✅ 1.0    | 0.9       | 0.8         | 0.9      | 0.9      | 0.88    |

Averages:
  has_answer: 1.00
  relevance: 0.90
  helpfulness: 0.88
  semantic_correctness: 0.83
  task_completion: 0.88
  overall: 0.87
```

---

## 10. Evaluation Pipeline

### 10.1 Automated Eval Script

```python
# eval_pipeline.py
"""รัน evaluation อัตโนมัติ — ใช้ใน CI/CD ได้"""

import sys
from langsmith import evaluate, Client

client = Client()

PASSING_THRESHOLD = 0.7  # ต้องได้ >= 70% ถึงผ่าน

def run_evaluation():
    results = evaluate(
        my_rag_bot,
        data="comprehensive-eval",
        evaluators=[relevance_eval, semantic_correctness_judge],
        experiment_prefix="ci-eval",
    )

    # ดึงผลลัพธ์
    scores = {}
    for result in results:
        for eval_result in result.get("evaluation_results", []):
            key = eval_result["key"]
            if key not in scores:
                scores[key] = []
            scores[key].append(eval_result["score"])

    # คำนวณค่าเฉลี่ย
    print("\n📊 Evaluation Results:")
    all_pass = True
    for key, values in scores.items():
        avg = sum(values) / len(values)
        status = "✅ PASS" if avg >= PASSING_THRESHOLD else "❌ FAIL"
        if avg < PASSING_THRESHOLD:
            all_pass = False
        print(f"  {key}: {avg:.2f} {status}")

    if not all_pass:
        print(f"\n❌ EVALUATION FAILED — some metrics below {PASSING_THRESHOLD}")
        sys.exit(1)
    else:
        print(f"\n✅ EVALUATION PASSED — all metrics >= {PASSING_THRESHOLD}")
        sys.exit(0)

if __name__ == "__main__":
    run_evaluation()
```

```bash
# ใน CI/CD
python eval_pipeline.py
# exit code 0 = pass, 1 = fail
```

---

## 11. ข้อจำกัดและ Pitfalls

### 11.1 Bias ของ LLM Judge

```
⚠️ Position Bias:
  - LLM มักให้คะแนน "คำตอบแรก" สูงกว่า
  - แก้: สลับตำแหน่ง A/B แล้วรัน 2 ครั้ง

⚠️ Verbosity Bias:
  - LLM มักให้คะแนน "คำตอบยาว" สูงกว่า
  - แก้: ระบุใน criteria ว่า "ความยาวไม่ใช่เกณฑ์"

⚠️ Self-preference Bias:
  - LLM มักให้คะแนน output ของ model เดียวกันสูงกว่า
  - แก้: ใช้ judge model คนละตัวกับ target model

⚠️ Inconsistency:
  - รัน 2 ครั้งได้คะแนนต่างกัน
  - แก้: ใช้ temperature=0 สำหรับ judge, รันหลายครั้งแล้วเฉลี่ย
```

### 11.2 เทคนิคลด Bias

```python
# 1. ใช้ temperature=0
judge_model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

# 2. สลับตำแหน่งสำหรับ pairwise
# (ดูตัวอย่างใน Section 5.1)

# 3. ใช้ judge model ที่ต่างจาก target model
# target: gemini-flash → judge: gpt-4o (หรือ claude)

# 4. รันหลายครั้งแล้วเฉลี่ย
def robust_judge(run, example, num_runs=3):
    scores = []
    for _ in range(num_runs):
        result = relevance_judge(run, example)
        scores.append(result["score"])
    avg = sum(scores) / len(scores)
    return {"key": "relevance_robust", "score": avg}
```

---

## 12. Best Practices

```
📋 Judge Prompt:
  ✅ เกณฑ์ชัดเจน + rubric (คำอธิบายแต่ละระดับคะแนน)
  ✅ กำหนด output format ชัด (SCORE: / REASON:)
  ✅ ให้ตัวอย่าง (few-shot) สำหรับ criteria ที่ซับซ้อน
  ❌ อย่ากว้างเกินไป ("ดีไหม?" → ไม่ดี)
  ❌ อย่าให้คะแนนช่วงกว้าง (1-100 → ยากต่อ consistency)

📋 Judge Model:
  ✅ ใช้ temperature=0 เสมอ
  ✅ ใช้ model ที่แม่นยำ (อาจใช้ model ที่ดีกว่า target)
  ✅ ใช้ model ต่างตัวจาก target (ลด self-bias)
  ⚠️ Balance ค่าใช้จ่าย vs ความแม่นยำ

📋 Dataset:
  ✅ ครอบคลุมทุก use case (happy path + edge case)
  ✅ มี reference answer ที่ถูกต้อง
  ✅ Update dataset สม่ำเสมอ (เพิ่ม cases จาก production)
  ✅ มีอย่างน้อย 20-50 test cases

📋 Process:
  ✅ รัน eval ทุกครั้งที่เปลี่ยน prompt/model
  ✅ เก็บผล evaluation เป็น history (ดู trend)
  ✅ ใช้ใน CI/CD pipeline
  ✅ ทำ human review ควบคู่ (สุ่มตรวจ)
  ✅ Calibrate judge กับ human rating (ดูว่า LLM judge สอดคล้องกับคนไหม)
```
