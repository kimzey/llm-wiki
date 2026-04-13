---
title: "Step 7: Evaluation & Observability"
type: source
source_file: raw/notes/LangChain/step7-evaluation-tutorial.md
tags: [langchain, evaluation, observability, testing, llm-as-judge]
related: []
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/LangChain/step7-evaluation-tutorial.md|Original file]]

## สรุป

บทช่วยสอน Evaluation & Observability สอนวิธีวัดคุณภาพ AI, ดู Trace, และใช้ LLM-as-Judge เนื้อหาครอบคลุม Observability ฟรีด้วย Phoenix และ LangFuse, Rule-based Evaluation, LLM-as-Judge, และ Evaluation Pipeline อัตโนมัติ

## ประเด็นสำคัญ

### ทำไมต้อง Evaluate?
- **ไม่มี Evaluation**: "เปลี่ยน prompt แล้ว... ดีขึ้นหรือเปล่านะ?" → ไม่รู้! ใช้ความรู้สึก 😅
- **มี Evaluation**:
  - "prompt v2 ได้คะแนน 0.85 vs v1 ได้ 0.72 → ดีขึ้น 18%"
  - "Agent ตอบถูก 92% จาก 50 test cases"
  - "Gemini Flash ได้ 0.88, GPT-4o-mini ได้ 0.91 → GPT ดีกว่านิดหน่อย"
  → รู้ชัดเจน! ตัดสินใจจากข้อมูล 📊

### Observability ฟรี ด้วย Phoenix
```bash
pip install arize-phoenix openinference-instrumentation-langchain
```
```python
import phoenix as px
px.launch_app()  # เปิด browser ไปที่ http://localhost:6006

from openinference.instrumentation.langchain import LangChainInstrumentor
from phoenix.otel import register

tracer_provider = register(project_name="my-rag-app")
LangChainInstrumentor().instrument(tracer_provider=tracer_provider)
# หลังจากนี้ ทุก LangChain operation จะถูก trace อัตโนมัติ!
```
- จะเห็น: input, output, latency, tokens ทุก call

### Observability ฟรี ด้วย LangFuse
```bash
pip install langfuse
```
```python
from langfuse.callback import CallbackHandler
langfuse = CallbackHandler()

# ใช้โดยเพิ่ม callbacks
response = model.invoke("สวัสดี", config={"callbacks": [langfuse]})
```

### Rule-based Evaluation
- **contains_expected**: ตรวจว่าคำตอบมีคำที่คาดหวังไหม
  ```python
  def contains_expected(predicted: str, expected: str) -> float:
      return 1.0 if expected.lower() in predicted.lower() else 0.0
  ```
- **reasonable_length**: ตรวจว่าคำตอบยาวพอ (ไม่สั้นเกิน ไม่ยาวเกิน)

### LLM-as-Judge (ใช้ AI ตรวจ AI)
- Rule-based ตรวจได้แค่ "มีคำนี้ไหม"
- LLM-as-Judge ตรวจได้ว่า "ตอบตรงประเด็นไหม" "สุภาพไหม" "hallucinate ไหม"

**Evaluators หลัก**:
1. **Relevance** (ตอบตรงคำถามไหม):
   - 5 = ตอบตรงคำถาม ครบถ้วน
   - 4 = ตอบตรง แต่ขาดรายละเอียดเล็กน้อย
   - 3 = เกี่ยวข้องบ้าง ไม่ครบ
   - 2 = เกี่ยวข้องน้อย
   - 1 = ไม่ตอบคำถาม

2. **Faithfulness** (ตอบจาก context จริงไหม):
   - 5 = ทุกข้อมูลมาจาก Context
   - 3 = บางส่วนมาจาก Context บางส่วนแต่ง
   - 1 = แทบไม่มีข้อมูลจาก Context (Hallucination!)

3. **Helpfulness** (มีประโยชน์ไหม):
   - 5 = มีประโยชน์มาก นำไปใช้ได้ทันที
   - 3 = มีประโยชน์ปานกลาง
   - 1 = ไม่มีประโยชน์

### parse_score helper
```python
def parse_score(text: str) -> tuple[int, str]:
    """ดึง SCORE: และ REASON: จาก judge response"""
    score = 3  # default
    reason = ""
    for line in text.strip().split("\n"):
        if line.upper().startswith("SCORE:"):
            score = int(line.split(":")[-1].strip().split("/")[0])
            score = max(1, min(5, score))  # clamp ให้อยู่ในช่วง 1-5
        elif line.upper().startswith("REASON:"):
            reason = line.split(":", 1)[-1].strip()
    return score, reason
```

### Evaluation Pipeline อัตโนมัติ
```python
def run_full_evaluation(
    bot_function,       # function ที่จะทดสอบ
    test_cases: list,   # ชุดทดสอบ
    evaluators: list,   # list ของ evaluator functions
    experiment_name: str = "experiment",
) -> dict:
    """รัน evaluation แบบเต็ม"""
```
- รัน evaluation → สรุป → บันทึกผล JSON
- เปรียบเทียบ experiments ได้

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- **Temperature สำหรับ Judge**: ใช้ `temperature=0` เพื่อความสม่ำเสมอ
- **Test cases ควรครอบคลุม**:
  - คำถามปกติ
  - คำถามที่มีคำศัพท์เฉพาะ
  - คำถามนอกขอบเขต (เพื่อดูว่าปฏิเสธไหม)
- **คะแนน**:
  - 4/5 หรือ 5/5 = ดีมาก ✅
  - 3/5 = ปานกลาง ⚠️
  - 1/5 หรือ 2/5 = แย่ ❌

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อมูลที่ขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/ai-agent.md|AI Agent]]
- [[wiki/concepts/agentic-rag.md|Agentic RAG]]
