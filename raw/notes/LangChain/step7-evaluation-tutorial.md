# Step 7: Evaluation & Observability
## วัดคุณภาพ AI + ดู Trace + LLM-as-Judge

> Step นี้จะทำให้รู้ว่า AI ตอบดีแค่ไหน เปลี่ยน prompt แล้วดีขึ้นหรือแย่ลง

---

## 7.1 ทำไมต้อง Evaluate?

```
ไม่มี Evaluation:
  "เปลี่ยน prompt แล้ว... ดีขึ้นหรือเปล่านะ?"
  "Agent ตอบถูกกี่ %?"
  "เปลี่ยน model แล้วคุ้มไหม?"
  → ไม่รู้! ใช้ความรู้สึก 😅

มี Evaluation:
  "prompt v2 ได้คะแนน 0.85 vs v1 ได้ 0.72 → ดีขึ้น 18%"
  "Agent ตอบถูก 92% จาก 50 test cases"
  "Gemini Flash ได้ 0.88, GPT-4o-mini ได้ 0.91 → GPT ดีกว่านิดหน่อย"
  → รู้ชัดเจน! ตัดสินใจจากข้อมูล 📊
```

---

## 7.2 Observability ฟรี ด้วย Phoenix

```bash
pip install arize-phoenix openinference-instrumentation-langchain
```

```python
# file: 36_phoenix.py

# ==== เปิด Phoenix (Local Tracing Dashboard) ====
import phoenix as px
px.launch_app()
# 🚀 เปิด browser ไปที่ http://localhost:6006

# ==== เชื่อมกับ LangChain ====
from openinference.instrumentation.langchain import LangChainInstrumentor
from phoenix.otel import register

tracer_provider = register(project_name="my-rag-app")
LangChainInstrumentor().instrument(tracer_provider=tracer_provider)
# ▲ หลังจากนี้ ทุก LangChain operation จะถูก trace อัตโนมัติ!
# ไม่ต้องแก้ code อื่นเลย

# ==== ใช้ LangChain ตามปกติ ====
from langchain_google_genai import ChatGoogleGenerativeAI
model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")

response = model.invoke("สวัสดี")
print(response.content)

# → ไปดู trace ที่ http://localhost:6006
# จะเห็น: input, output, latency, tokens ทุก call
```

---

## 7.3 Observability ฟรี ด้วย LangFuse

```bash
pip install langfuse
```

```python
# file: 37_langfuse.py

import os
from langfuse.callback import CallbackHandler

# ตั้งค่า (ใช้ cloud free tier หรือ self-host)
os.environ["LANGFUSE_PUBLIC_KEY"] = "pk-..."
os.environ["LANGFUSE_SECRET_KEY"] = "sk-..."
os.environ["LANGFUSE_HOST"] = "https://cloud.langfuse.com"  # หรือ http://localhost:3000

langfuse = CallbackHandler()

# ใช้โดยเพิ่ม callbacks
response = model.invoke("สวัสดี", config={"callbacks": [langfuse]})

# RAG Chain ก็ใช้ได้
answer = rag_chain.invoke("ลาพักร้อนกี่วัน", config={"callbacks": [langfuse]})

# LangGraph ก็ใช้ได้
result = graph.invoke(
    {"messages": [{"role": "user", "content": "สวัสดี"}]},
    config={"callbacks": [langfuse]}
)
```

---

## 7.4 Evaluation แบบ Rule-based

```python
# file: 38_rule_eval.py

# ==== สร้างชุดทดสอบ ====
test_cases = [
    {
        "question": "ลาพักร้อนได้กี่วัน?",
        "expected": "15 วัน",
        "category": "leave",
    },
    {
        "question": "VPN server คืออะไร?",
        "expected": "vpn.xyz.com",
        "category": "it",
    },
    {
        "question": "ค่าอาหารวันละเท่าไหร่?",
        "expected": "100 บาท",
        "category": "benefits",
    },
    {
        "question": "ลาป่วยกี่วันต้องมีใบรับรองแพทย์?",
        "expected": "3 วัน",
        "category": "leave",
    },
    {
        "question": "วิธีทำส้มตำ?",
        "expected": "ไม่มีข้อมูล",
        "category": "out_of_scope",
    },
]

# ==== สร้าง Evaluators ====

def contains_expected(predicted: str, expected: str) -> float:
    """ตรวจว่าคำตอบมีคำที่คาดหวังไหม"""
    # ▲ Rule-based = ตรวจด้วยกฎ ไม่ต้องใช้ AI
    return 1.0 if expected.lower() in predicted.lower() else 0.0

def reasonable_length(predicted: str) -> float:
    """ตรวจว่าคำตอบยาวพอ (ไม่สั้นเกิน ไม่ยาวเกิน)"""
    word_count = len(predicted.split())
    if 5 <= word_count <= 200:
        return 1.0
    return 0.0

# ==== รัน Evaluation ====
results = []

for tc in test_cases:
    # เรียก AI
    answer = ask(tc["question"])  # ← ใช้ RAG function จาก Step 3

    # ให้คะแนน
    score_contains = contains_expected(answer, tc["expected"])
    score_length = reasonable_length(answer)

    results.append({
        "question": tc["question"],
        "expected": tc["expected"],
        "predicted": answer[:80],
        "contains": score_contains,
        "length_ok": score_length,
    })

    status = "✅" if score_contains == 1.0 else "❌"
    print(f"{status} Q: {tc['question']}")
    print(f"   Expected: '{tc['expected']}' | Found: {score_contains == 1.0}")
    print(f"   Answer: {answer[:80]}...\n")

# ==== สรุป ====
avg_contains = sum(r["contains"] for r in results) / len(results)
avg_length = sum(r["length_ok"] for r in results) / len(results)
print(f"📊 Results:")
print(f"   Contains expected: {avg_contains:.0%}")
print(f"   Reasonable length: {avg_length:.0%}")
```

**Output:**
```
✅ Q: ลาพักร้อนได้กี่วัน?
   Expected: '15 วัน' | Found: True
   Answer: พนักงานมีสิทธิ์ลาพักร้อน 15 วันทำการต่อปี...

✅ Q: VPN server คืออะไร?
   Expected: 'vpn.xyz.com' | Found: True
   Answer: VPN server คือ vpn.xyz.com ใช้กับ FortiClient...

✅ Q: ค่าอาหารวันละเท่าไหร่?
   Expected: '100 บาท' | Found: True
   Answer: ค่าอาหาร 100 บาทต่อวันทำการ...

✅ Q: ลาป่วยกี่วันต้องมีใบรับรองแพทย์?
   Expected: '3 วัน' | Found: True
   Answer: ลาป่วยเกิน 3 วันติดต่อกันต้องมีใบรับรองแพทย์...

✅ Q: วิธีทำส้มตำ?
   Expected: 'ไม่มีข้อมูล' | Found: True
   Answer: ขออภัย ไม่มีข้อมูลเรื่องนี้ในเอกสาร...

📊 Results:
   Contains expected: 100%
   Reasonable length: 100%
```

---

## 7.5 LLM-as-Judge — ใช้ AI ตรวจ AI

```python
# file: 39_llm_judge.py

# Rule-based ตรวจได้แค่ "มีคำนี้ไหม"
# แต่ LLM-as-Judge ตรวจได้ว่า "ตอบตรงประเด็นไหม" "สุภาพไหม" "hallucinate ไหม"

judge_model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

def parse_score(text: str) -> tuple[int, str]:
    """ดึง SCORE: และ REASON: จาก judge response"""
    score = 3  # default
    reason = ""
    for line in text.strip().split("\n"):
        if line.upper().startswith("SCORE:"):
            try:
                score = int(line.split(":")[-1].strip().split("/")[0])
                score = max(1, min(5, score))
                #       ▲
                #  clamp ให้อยู่ในช่วง 1-5
            except:
                pass
        elif line.upper().startswith("REASON:"):
            reason = line.split(":", 1)[-1].strip()
    return score, reason

# ==== Evaluator 1: Relevance (ตอบตรงคำถามไหม) ====
def judge_relevance(question: str, answer: str) -> dict:
    prompt = f"""ให้คะแนน "ความเกี่ยวข้อง" ของคำตอบ:

เกณฑ์:
  5 = ตอบตรงคำถาม ครบถ้วน
  4 = ตอบตรง แต่ขาดรายละเอียดเล็กน้อย
  3 = เกี่ยวข้องบ้าง ไม่ครบ
  2 = เกี่ยวข้องน้อย
  1 = ไม่ตอบคำถาม

คำถาม: {question}
คำตอบ: {answer}

SCORE: <1-5>
REASON: <เหตุผลสั้นๆ>"""

    response = judge_model.invoke(prompt)
    score, reason = parse_score(response.content)
    return {"metric": "relevance", "score": score, "reason": reason}

# ==== Evaluator 2: Faithfulness (ตอบจาก context จริงไหม) ====
def judge_faithfulness(question: str, answer: str, context: str) -> dict:
    prompt = f"""ตรวจว่าคำตอบมาจาก Context จริงหรือแต่งเพิ่ม:

Context: {context[:500]}
คำตอบ: {answer}

เกณฑ์:
  5 = ทุกข้อมูลมาจาก Context
  3 = บางส่วนมาจาก Context บางส่วนแต่ง
  1 = แทบไม่มีข้อมูลจาก Context (Hallucination!)

SCORE: <1-5>
REASON: <ระบุส่วนที่ hallucinate ถ้ามี>"""

    response = judge_model.invoke(prompt)
    score, reason = parse_score(response.content)
    return {"metric": "faithfulness", "score": score, "reason": reason}

# ==== Evaluator 3: Helpfulness (มีประโยชน์ไหม) ====
def judge_helpfulness(question: str, answer: str) -> dict:
    prompt = f"""ให้คะแนน "ความเป็นประโยชน์":

  5 = มีประโยชน์มาก นำไปใช้ได้ทันที
  3 = มีประโยชน์ปานกลาง
  1 = ไม่มีประโยชน์

คำถาม: {question}
คำตอบ: {answer}

SCORE: <1-5>
REASON: <เหตุผล>"""

    response = judge_model.invoke(prompt)
    score, reason = parse_score(response.content)
    return {"metric": "helpfulness", "score": score, "reason": reason}

# ==== รัน LLM-as-Judge Evaluation ====
print("📊 LLM-as-Judge Evaluation\n")

all_scores = {"relevance": [], "helpfulness": [], "faithfulness": []}

for tc in test_cases:
    answer = ask(tc["question"])

    r = judge_relevance(tc["question"], answer)
    h = judge_helpfulness(tc["question"], answer)

    all_scores["relevance"].append(r["score"])
    all_scores["helpfulness"].append(h["score"])

    print(f"❓ {tc['question']}")
    print(f"   Relevance:   {r['score']}/5 — {r['reason']}")
    print(f"   Helpfulness: {h['score']}/5 — {h['reason']}")
    print()

# สรุป
print("=" * 50)
print("📊 Summary:")
for metric, scores in all_scores.items():
    avg = sum(scores) / len(scores)
    print(f"   {metric}: {avg:.1f}/5 ({avg/5:.0%})")
```

**Output:**
```
📊 LLM-as-Judge Evaluation

❓ ลาพักร้อนได้กี่วัน?
   Relevance:   5/5 — ตอบตรงคำถาม ระบุจำนวนวันชัดเจน
   Helpfulness: 5/5 — ให้ข้อมูลครบ สามารถนำไปใช้ได้ทันที

❓ VPN server คืออะไร?
   Relevance:   5/5 — ตอบ server address ได้ถูกต้อง
   Helpfulness: 5/5 — ระบุ address และวิธีใช้งานด้วย

❓ วิธีทำส้มตำ?
   Relevance:   5/5 — ปฏิเสธอย่างถูกต้อง บอกว่าไม่มีข้อมูล
   Helpfulness: 4/5 — ปฏิเสธสุภาพ แต่ไม่ได้แนะนำทางเลือก

==================================================
📊 Summary:
   relevance: 4.8/5 (96%)
   helpfulness: 4.6/5 (92%)
```

---

## 7.6 Evaluation Pipeline อัตโนมัติ

```python
# file: 40_eval_pipeline.py

import json
from datetime import datetime

def run_full_evaluation(
    bot_function,       # function ที่จะทดสอบ
    test_cases: list,   # ชุดทดสอบ
    evaluators: list,   # list ของ evaluator functions
    experiment_name: str = "experiment",
) -> dict:
    """รัน evaluation แบบเต็ม"""

    results = []
    all_scores = {}

    print(f"🧪 Running: {experiment_name}")
    print(f"📝 Test cases: {len(test_cases)}")
    print(f"📏 Evaluators: {len(evaluators)}")
    print("=" * 50)

    for i, tc in enumerate(test_cases, 1):
        answer = bot_function(tc["question"])

        scores = {}
        for evaluator in evaluators:
            result = evaluator(tc["question"], answer)
            metric = result["metric"]
            scores[metric] = result["score"]

            if metric not in all_scores:
                all_scores[metric] = []
            all_scores[metric].append(result["score"])

        results.append({
            "question": tc["question"],
            "expected": tc.get("expected", ""),
            "predicted": answer,
            "scores": scores,
        })

        # แสดงผลทีละข้อ
        status = "✅" if all(s >= 4 for s in scores.values()) else "⚠️"
        scores_str = " | ".join([f"{k}: {v}/5" for k, v in scores.items()])
        print(f"  {status} [{i}/{len(test_cases)}] {tc['question'][:40]}... → {scores_str}")

    # สรุป
    print("\n" + "=" * 50)
    print(f"📊 Summary: {experiment_name}")
    summary = {}
    for metric, scores in all_scores.items():
        avg = sum(scores) / len(scores)
        summary[metric] = avg
        status = "✅" if avg >= 4 else "⚠️" if avg >= 3 else "❌"
        print(f"   {status} {metric}: {avg:.1f}/5 ({avg/5:.0%})")

    # บันทึกผลลัพธ์
    output = {
        "experiment": experiment_name,
        "timestamp": datetime.now().isoformat(),
        "num_test_cases": len(test_cases),
        "summary": summary,
        "details": results,
    }

    filename = f"eval_{experiment_name}_{datetime.now().strftime('%Y%m%d_%H%M')}.json"
    with open(filename, "w", encoding="utf-8") as f:
        json.dump(output, f, ensure_ascii=False, indent=2)
    print(f"\n💾 Saved to: {filename}")

    return output

# ==== ใช้งาน ====

# ทดสอบ RAG Bot
run_full_evaluation(
    bot_function=ask,       # RAG function จาก Step 3
    test_cases=test_cases,
    evaluators=[judge_relevance, judge_helpfulness],
    experiment_name="rag-v1-gemini-flash",
)

# เปลี่ยน prompt แล้วทดสอบอีกรอบ → เปรียบเทียบ!
# run_full_evaluation(ask_v2, test_cases, evaluators, "rag-v2-new-prompt")
```

**Output:**
```
🧪 Running: rag-v1-gemini-flash
📝 Test cases: 5
📏 Evaluators: 2
==================================================
  ✅ [1/5] ลาพักร้อนได้กี่วัน?... → relevance: 5/5 | helpfulness: 5/5
  ✅ [2/5] VPN server คืออะไร?... → relevance: 5/5 | helpfulness: 5/5
  ✅ [3/5] ค่าอาหารวันละเท่าไหร่?... → relevance: 5/5 | helpfulness: 4/5
  ✅ [4/5] ลาป่วยกี่วันต้องมีใบรับรอง... → relevance: 5/5 | helpfulness: 5/5
  ✅ [5/5] วิธีทำส้มตำ?... → relevance: 5/5 | helpfulness: 4/5

==================================================
📊 Summary: rag-v1-gemini-flash
   ✅ relevance: 5.0/5 (100%)
   ✅ helpfulness: 4.6/5 (92%)

💾 Saved to: eval_rag-v1-gemini-flash_20260410_1430.json
```

---

## สรุป Step 7

```
✅ Observability ฟรีด้วย Phoenix (pip install แล้วใช้เลย)
✅ Observability ฟรีด้วย LangFuse (ใช้ callbacks)
✅ Rule-based Evaluation (contains, length check)
✅ LLM-as-Judge (relevance, faithfulness, helpfulness)
✅ parse_score helper สำหรับดึงคะแนนจาก Judge
✅ Evaluation Pipeline อัตโนมัติ พร้อมบันทึกผล JSON
✅ เปรียบเทียบ experiments ได้
```

### ไป Step 8 → Deployment (Deploy ขึ้น Cloud)
