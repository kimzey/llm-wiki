# Step 6: Multi-Agent Systems
## สร้าง Agent หลายตัวทำงานร่วมกัน

> Step นี้จะสร้าง "ทีม AI" ที่แต่ละตัวมีความเชี่ยวชาญเฉพาะทาง

---

## 6.1 ทำไมต้องหลาย Agent?

```
Single Agent (Agent เดียว):
  System prompt: "คุณเป็นทั้ง HR expert, IT support, นักเขียน, นักวิจัย, บรรณาธิการ..."
  ❌ prompt ยาวมาก → สับสน
  ❌ ยากต่อการ debug
  ❌ ทำงานทีละอย่าง

Multi-Agent (หลาย Agent):
  Agent 1: "คุณเป็น HR expert"          ← prompt สั้น ชัดเจน
  Agent 2: "คุณเป็น IT support"         ← แยก role
  Agent 3: "คุณเป็นหัวหน้าทีม"          ← ตัดสินใจว่าส่งให้ใคร
  ✅ แต่ละตัวเก่งในสิ่งที่ถนัด
  ✅ debug ง่าย
  ✅ ทำงานขนานได้
```

---

## 6.2 Pattern 1: Supervisor — มีหัวหน้าสั่งงาน

```python
# file: 33_supervisor.py

from dotenv import load_dotenv
from typing import Annotated, Literal
from typing_extensions import TypedDict
from langchain_google_genai import ChatGoogleGenerativeAI
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from pydantic import BaseModel, Field

load_dotenv()

# ==== State ====
class TeamState(TypedDict):
    messages: Annotated[list, add_messages]
    next_agent: str  # ← Supervisor จะตั้งค่านี้

# ==== Models ====
boss_model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)
worker_model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0.3)

# ==== Supervisor (หัวหน้า) ====

# ใช้ Structured Output เพื่อให้ routing แม่นยำ
class Decision(BaseModel):
    next: str = Field(description="ส่งให้ใคร: researcher, writer, reviewer, FINISH")
    reason: str = Field(description="เหตุผลสั้นๆ")

def supervisor(state: TeamState):
    """หัวหน้า: ดู state แล้วตัดสินใจว่าส่งให้ใคร"""
    structured = boss_model.with_structured_output(Decision)
    #            ▲
    #  ใช้ structured output เพื่อให้ได้ JSON ที่ parse ได้
    #  ไม่ใช่ free text ที่อาจ parse ผิด

    decision = structured.invoke([
        {"role": "system", "content": """คุณคือหัวหน้าทีม สมาชิก:
- researcher: ค้นหาข้อมูล (ส่งเมื่อยังไม่มีข้อมูล)
- writer: เขียนเนื้อหา (ส่งเมื่อมีข้อมูลแล้ว)
- reviewer: ตรวจสอบ (ส่งเมื่อเขียนแล้ว)
- FINISH: จบงาน (ส่งเมื่อ reviewer อนุมัติ)"""},
        *state["messages"]
    ])

    print(f"👔 Supervisor → {decision.next} ({decision.reason})")
    return {"next_agent": decision.next}

# ==== Workers ====
def researcher(state: TeamState):
    """นักวิจัย: ค้นหาข้อมูล"""
    response = worker_model.invoke([
        {"role": "system", "content": "คุณคือนักวิจัย รวบรวมข้อมูลอย่างละเอียด"},
        *state["messages"]
    ])
    print(f"🔬 Researcher: {response.content[:60]}...")
    return {"messages": [{"role": "assistant", "content": f"[Researcher] {response.content}"}]}

def writer(state: TeamState):
    """นักเขียน: เรียบเรียงเนื้อหา"""
    response = worker_model.invoke([
        {"role": "system", "content": "คุณคือนักเขียน เรียบเรียงข้อมูลเป็นบทความอ่านง่าย"},
        *state["messages"]
    ])
    print(f"✍️  Writer: {response.content[:60]}...")
    return {"messages": [{"role": "assistant", "content": f"[Writer] {response.content}"}]}

def reviewer(state: TeamState):
    """บรรณาธิการ: ตรวจสอบ"""
    response = worker_model.invoke([
        {"role": "system", "content": """คุณคือบรรณาธิการ ตรวจบทความ
ถ้าดี: ตอบ "APPROVED: (สรุป)"
ถ้าต้องแก้: บอก feedback"""},
        *state["messages"]
    ])
    print(f"📝 Reviewer: {response.content[:60]}...")
    return {"messages": [{"role": "assistant", "content": f"[Reviewer] {response.content}"}]}

# ==== Routing ====
def route(state: TeamState) -> str:
    next_a = state.get("next_agent", "FINISH")
    if next_a in ["researcher", "writer", "reviewer"]:
        return next_a
    return END
    # return END = ไปที่จุดจบ

# ==== Build Graph ====
builder = StateGraph(TeamState)

builder.add_node("supervisor", supervisor)
builder.add_node("researcher", researcher)
builder.add_node("writer", writer)
builder.add_node("reviewer", reviewer)

builder.add_edge(START, "supervisor")
builder.add_conditional_edges("supervisor", route)

# ทุก worker → กลับไป supervisor
builder.add_edge("researcher", "supervisor")
builder.add_edge("writer", "supervisor")
builder.add_edge("reviewer", "supervisor")

team = builder.compile()

# ==== ทดสอบ ====
result = team.invoke({
    "messages": [{"role": "user", "content": "เขียนบทความสั้นเรื่อง RAG คืออะไร"}],
    "next_agent": "",
})

print("\n📄 บทความสุดท้าย:")
# หา message สุดท้ายจาก Writer
for msg in reversed(result["messages"]):
    if hasattr(msg, "content") and "[Writer]" in str(msg.content):
        print(msg.content.replace("[Writer] ", ""))
        break
```

**Output:**
```
👔 Supervisor → researcher (ยังไม่มีข้อมูล ต้องค้นหาก่อน)
🔬 Researcher: RAG (Retrieval-Augmented Generation) คือเทคนิคที่...
👔 Supervisor → writer (มีข้อมูลแล้ว ให้เขียน)
✍️  Writer: # RAG คืออะไร? ...
👔 Supervisor → reviewer (เขียนแล้ว ต้องตรวจ)
📝 Reviewer: APPROVED: บทความครอบคลุมดี อ่านเข้าใจง่าย
👔 Supervisor → FINISH (reviewer อนุมัติแล้ว)
```

---

## 6.3 Pattern 2: Handoff — ส่งต่อตามความเชี่ยวชาญ

```python
# file: 34_handoff.py

# แนวคิด:
# User ถามมา → Triage Agent รับเรื่อง → ส่งต่อให้ Agent ที่เชี่ยวชาญ
# เหมือนโทรไป Call Center แล้วถูกโอนสาย

class HandoffState(TypedDict):
    messages: Annotated[list, add_messages]
    department: str

def triage(state: HandoffState):
    """รับเรื่อง แล้วจำแนกว่าส่งแผนกไหน"""
    response = boss_model.invoke([
        {"role": "system", "content": """จำแนกคำถาม:
- "hr" = เรื่องลา, สวัสดิการ, เงินเดือน
- "it" = เรื่องระบบ, รหัสผ่าน, VPN
- "general" = อื่นๆ
ตอบแค่คำเดียว: hr, it, general"""},
        state["messages"][-1]
    ])
    dept = response.content.strip().lower()
    if dept not in ["hr", "it", "general"]:
        dept = "general"
    print(f"📞 Triage → แผนก {dept}")
    return {"department": dept}

def hr_agent(state: HandoffState):
    """แผนก HR"""
    response = worker_model.invoke([
        {"role": "system", "content": "คุณคือ HR ตอบเรื่องลา สวัสดิการ นโยบาย"},
        *state["messages"]
    ])
    return {"messages": [response]}

def it_agent(state: HandoffState):
    """แผนก IT"""
    response = worker_model.invoke([
        {"role": "system", "content": "คุณคือ IT Support ตอบเรื่องระบบ รหัสผ่าน VPN"},
        *state["messages"]
    ])
    return {"messages": [response]}

def general_agent(state: HandoffState):
    """ทั่วไป"""
    response = worker_model.invoke([
        {"role": "system", "content": "คุณคือผู้ช่วยทั่วไป"},
        *state["messages"]
    ])
    return {"messages": [response]}

def route_dept(state: HandoffState) -> Literal["hr_agent", "it_agent", "general_agent"]:
    return {"hr": "hr_agent", "it": "it_agent"}.get(state["department"], "general_agent")

# Build
builder = StateGraph(HandoffState)
builder.add_node("triage", triage)
builder.add_node("hr_agent", hr_agent)
builder.add_node("it_agent", it_agent)
builder.add_node("general_agent", general_agent)

builder.add_edge(START, "triage")
builder.add_conditional_edges("triage", route_dept)
builder.add_edge("hr_agent", END)
builder.add_edge("it_agent", END)
builder.add_edge("general_agent", END)

handoff = builder.compile()

# ทดสอบ
for q in ["ลาพักร้อนกี่วัน", "ลืมรหัสผ่าน", "ร้านอาหารแนะนำ"]:
    r = handoff.invoke({"messages": [{"role": "user", "content": q}], "department": ""})
    print(f"  ❓ {q}")
    print(f"  💬 {r['messages'][-1].content[:80]}...\n")
```

---

## 6.4 Pattern 3: Pipeline — สายพานผลิต

```python
# file: 35_pipeline.py

# แนวคิด: A → B → C → D (ทำงานต่อเนื่องเป็นขั้นตอน)

class PipelineState(TypedDict):
    messages: Annotated[list, add_messages]
    original: str
    translated: str
    summary: str

def translator(state: PipelineState):
    r = worker_model.invoke(f"แปลเป็นอังกฤษ ตอบแค่คำแปล:\n{state['original']}")
    print(f"🌐 Translated")
    return {"translated": r.content}

def summarizer(state: PipelineState):
    r = worker_model.invoke(f"สรุปเป็น 1 ประโยค:\n{state['translated']}")
    print(f"📝 Summarized")
    return {"summary": r.content}

def formatter(state: PipelineState):
    output = f"📄 Original: {state['original']}\n🌐 English: {state['translated']}\n📝 Summary: {state['summary']}"
    print(f"🎨 Formatted")
    return {"messages": [{"role": "assistant", "content": output}]}

builder = StateGraph(PipelineState)
builder.add_node("translate", translator)
builder.add_node("summarize", summarizer)
builder.add_node("format", formatter)
builder.add_edge(START, "translate")
builder.add_edge("translate", "summarize")
builder.add_edge("summarize", "format")
builder.add_edge("format", END)

pipeline = builder.compile()

result = pipeline.invoke({
    "messages": [],
    "original": "LangGraph เป็น library สำหรับสร้าง agent workflow ที่ซับซ้อน",
    "translated": "", "summary": ""
})
print(result["messages"][-1].content)
```

**Output:**
```
🌐 Translated
📝 Summarized
🎨 Formatted
📄 Original: LangGraph เป็น library สำหรับสร้าง agent workflow ที่ซับซ้อน
🌐 English: LangGraph is a library for building complex agent workflows.
📝 Summary: LangGraph enables complex AI agent workflow creation.
```

---

## สรุป Step 6

```
✅ Supervisor Pattern — หัวหน้าสั่งงาน Worker
✅ Handoff Pattern — Triage แล้วส่งต่อแผนกที่เชี่ยวชาญ
✅ Pipeline Pattern — สายพานทำงานต่อเนื่อง
✅ ใช้ Structured Output สำหรับ routing ที่แม่นยำ
✅ เข้าใจว่าเมื่อไหร่ควรใช้ pattern ไหน
```

### ไป Step 7 → Evaluation (วัดคุณภาพ AI)
