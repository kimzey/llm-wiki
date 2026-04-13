# Multi-Agent Systems Deep Dive
## สร้าง Agent หลายตัวทำงานร่วมกันผ่าน LangGraph

> Multi-Agent = หลาย Agent แต่ละตัวมีความเชี่ยวชาญเฉพาะทาง ทำงานร่วมกันเพื่อทำงานที่ซับซ้อน

---

## สารบัญ

1. [ทำไมต้อง Multi-Agent](#1-ทำไมต้อง-multi-agent)
2. [Architecture Patterns](#2-architecture-patterns)
3. [Pattern 1: Supervisor (มีหัวหน้า)](#3-supervisor-pattern)
4. [Pattern 2: Handoff (ส่งต่อ)](#4-handoff-pattern)
5. [Pattern 3: Swarm (ฝูง)](#5-swarm-pattern)
6. [Pattern 4: Pipeline (สายพาน)](#6-pipeline-pattern)
7. [Pattern 5: Debate (ถกเถียง)](#7-debate-pattern)
8. [Shared State & Communication](#8-shared-state)
9. [ตัวอย่าง: Content Creation Team](#9-content-creation-team)
10. [ตัวอย่าง: Software Development Team](#10-software-dev-team)
11. [ตัวอย่าง: Customer Support Escalation](#11-customer-support)
12. [Error Handling & Safeguards](#12-error-handling)
13. [Best Practices](#13-best-practices)

---

## 1. ทำไมต้อง Multi-Agent

### Single Agent vs Multi-Agent

```
Single Agent:
  ❌ System prompt ยาวมาก (หลาย role ในตัวเดียว)
  ❌ เปลี่ยน model ไม่ได้ (บาง task ต้องการ model ที่ต่างกัน)
  ❌ ยากต่อการ debug (ทุกอย่างรวมกันหมด)
  ❌ ไม่สามารถทำงานขนานได้

Multi-Agent:
  ✅ แต่ละ Agent มี role ชัดเจน prompt สั้นกระชับ
  ✅ ใช้ model ที่เหมาะสมกับแต่ละ task
  ✅ Debug ง่าย (แยก trace ตาม Agent)
  ✅ ทำงานขนานได้ (Parallel execution)
  ✅ เพิ่ม/ลด Agent ได้ง่าย (Modular)
```

### เมื่อไหร่ควรใช้ Multi-Agent

```
✅ ใช้เมื่อ:
  - งานมีหลายขั้นตอนที่ต้องใช้ความเชี่ยวชาญต่างกัน
  - ต้องการ quality control (Agent ตรวจ Agent)
  - ต้องการ parallel processing
  - ระบบซับซ้อนเกินกว่า single agent จะจัดการได้

❌ ไม่จำเป็นเมื่อ:
  - งาน simple ที่ Agent เดียวทำได้
  - ไม่ต้องการ role แยกชัดเจน
  - ต้นทุน/latency เป็นปัจจัยหลัก
```

---

## 2. Architecture Patterns

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│  1. Supervisor     2. Handoff      3. Swarm         │
│  ┌───┐            A ──→ B         A ↔ B ↔ C        │
│  │ S │            ↓     ↓           ↕   ↕          │
│  └┬┬┬┘            C ──→ D         D ↔ E ↔ F        │
│   ││└→ C                                            │
│   │└→ B                                             │
│   └→ A                                              │
│                                                     │
│  4. Pipeline       5. Debate                        │
│  A → B → C → D    A ←→ B                           │
│  (สายพาน)         Judge ↓                           │
│                   (ตัดสิน)                           │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 3. Supervisor Pattern

### 3.1 แนวคิด

มี "Supervisor" Agent เป็นหัวหน้า คอยตัดสินใจว่าจะสั่งให้ Agent ไหนทำงาน:

```
User → Supervisor → "ให้ Researcher ไปค้นข้อมูล"
                  → "ให้ Writer เขียนบทความ"
                  → "ให้ Reviewer ตรวจสอบ"
                  → "เสร็จแล้ว ส่งผลลัพธ์"
```

### 3.2 Implementation

```python
import os
from dotenv import load_dotenv
from typing import Annotated, Literal
from typing_extensions import TypedDict
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain.tools import tool
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from pydantic import BaseModel, Field

load_dotenv()

# ======== State ========
class TeamState(TypedDict):
    messages: Annotated[list, add_messages]
    next_agent: str
    task: str
    final_output: str

# ======== Models ========
supervisor_model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)
worker_model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0.5)

# ======== Supervisor ========
TEAM_MEMBERS = ["researcher", "writer", "reviewer"]

class SupervisorDecision(BaseModel):
    next: str = Field(description=f"Agent ถัดไป: {', '.join(TEAM_MEMBERS)} หรือ FINISH")
    reason: str = Field(description="เหตุผลสั้นๆ")

def supervisor(state: TeamState):
    """หัวหน้าทีม ตัดสินใจว่าจะสั่ง Agent ไหนทำงาน"""
    structured_model = supervisor_model.with_structured_output(SupervisorDecision)

    messages = [
        {"role": "system", "content": f"""คุณคือหัวหน้าทีมที่คอยสั่งงาน
สมาชิกในทีม:
- researcher: ค้นหาและรวบรวมข้อมูล
- writer: เขียนเนื้อหาจากข้อมูลที่ได้
- reviewer: ตรวจสอบคุณภาพและให้ feedback

ดูประวัติการทำงานแล้วตัดสินใจ:
- ถ้ายังไม่มีข้อมูล → ส่งให้ researcher
- ถ้ามีข้อมูลแล้ว แต่ยังไม่ได้เขียน → ส่งให้ writer
- ถ้าเขียนแล้ว แต่ยังไม่ตรวจ → ส่งให้ reviewer
- ถ้า reviewer อนุมัติแล้ว → FINISH
- ถ้า reviewer ให้แก้ → ส่งให้ writer อีกรอบ"""},
        *state["messages"]
    ]

    decision = structured_model.invoke(messages)
    print(f"👔 Supervisor: → {decision.next} ({decision.reason})")

    return {"next_agent": decision.next}

# ======== Workers ========
def researcher(state: TeamState):
    """นักวิจัย ค้นหาข้อมูล"""
    messages = [
        {"role": "system", "content": "คุณคือนักวิจัย ค้นหาและรวบรวมข้อมูลอย่างละเอียด ระบุแหล่งที่มา"},
        *state["messages"]
    ]
    response = worker_model.invoke(messages)
    print(f"🔬 Researcher: {response.content[:80]}...")
    return {"messages": [{"role": "assistant", "content": f"[Researcher]: {response.content}"}]}

def writer(state: TeamState):
    """นักเขียน เรียบเรียงเนื้อหา"""
    messages = [
        {"role": "system", "content": "คุณคือนักเขียน นำข้อมูลที่ Researcher หามามาเรียบเรียงเป็นบทความที่อ่านง่าย"},
        *state["messages"]
    ]
    response = worker_model.invoke(messages)
    print(f"✍️  Writer: {response.content[:80]}...")
    return {"messages": [{"role": "assistant", "content": f"[Writer]: {response.content}"}]}

def reviewer(state: TeamState):
    """บรรณาธิการ ตรวจสอบ"""
    messages = [
        {"role": "system", "content": """คุณคือบรรณาธิการ ตรวจสอบบทความของ Writer
- ถ้าดีแล้ว: ตอบว่า "APPROVED: <สรุปสั้นๆ>"
- ถ้าต้องแก้: บอก feedback ชัดเจน"""},
        *state["messages"]
    ]
    response = worker_model.invoke(messages)
    print(f"📝 Reviewer: {response.content[:80]}...")
    return {"messages": [{"role": "assistant", "content": f"[Reviewer]: {response.content}"}]}

# ======== Routing ========
def route_supervisor(state: TeamState) -> str:
    next_agent = state.get("next_agent", "FINISH")
    if next_agent == "FINISH" or next_agent not in TEAM_MEMBERS:
        return END
    return next_agent

# ======== Build Graph ========
builder = StateGraph(TeamState)

builder.add_node("supervisor", supervisor)
builder.add_node("researcher", researcher)
builder.add_node("writer", writer)
builder.add_node("reviewer", reviewer)

# ทุก path เริ่มที่ supervisor
builder.add_edge(START, "supervisor")

# Supervisor ตัดสินใจว่าส่งให้ใคร
builder.add_conditional_edges("supervisor", route_supervisor)

# หลังแต่ละ worker ทำเสร็จ กลับไปให้ supervisor ตัดสินใจอีกรอบ
builder.add_edge("researcher", "supervisor")
builder.add_edge("writer", "supervisor")
builder.add_edge("reviewer", "supervisor")

team = builder.compile()

# ======== ทดสอบ ========
result = team.invoke({
    "messages": [{"role": "user", "content": "เขียนบทความสั้นเรื่อง 'ทำไม RAG ถึงสำคัญ'"}],
    "next_agent": "",
    "task": "",
    "final_output": "",
})

# ดู flow
print("\n📋 Full Flow:")
for msg in result["messages"]:
    if hasattr(msg, 'content'):
        content = msg.content if isinstance(msg.content, str) else str(msg.content)
        if content.startswith("["):
            agent = content.split("]")[0] + "]"
            print(f"  {agent} {content[len(agent):len(agent)+80]}...")
```

**Output:**
```
👔 Supervisor: → researcher (ยังไม่มีข้อมูล ต้องค้นหาก่อน)
🔬 Researcher: RAG (Retrieval-Augmented Generation) คือเทคนิคที่...
👔 Supervisor: → writer (มีข้อมูลแล้ว ให้เขียนบทความ)
✍️  Writer: # ทำไม RAG ถึงสำคัญ? ในโลกของ AI ปัจจุบัน...
👔 Supervisor: → reviewer (บทความเขียนแล้ว ต้องตรวจ)
📝 Reviewer: APPROVED: บทความครอบคลุมดี มีตัวอย่างชัดเจน...
👔 Supervisor: → FINISH (reviewer อนุมัติแล้ว)
```

---

## 4. Handoff Pattern

### 4.1 แนวคิด

Agent ส่งต่อ (handoff) งานให้ Agent อื่นโดยตรง ไม่ผ่าน supervisor:

```
User → Triage Agent → (billing?) → Billing Agent → Answer
                    → (tech?)    → Tech Agent → (ยากเกิน?) → Senior Tech → Answer
                    → (general?) → General Agent → Answer
```

### 4.2 Implementation

```python
from langgraph.prebuilt import create_react_agent
from langgraph.types import Command

class HandoffState(TypedDict):
    messages: Annotated[list, add_messages]
    current_agent: str

# สร้าง Handoff tools
@tool
def transfer_to_billing():
    """โอนสายให้แผนกการเงิน สำหรับคำถามเกี่ยวกับ ค่าใช้จ่าย ใบเสร็จ การชำระเงิน"""
    pass  # Tool นี้ไม่ได้ทำอะไรจริง แค่เป็น signal ให้ router

@tool
def transfer_to_tech():
    """โอนสายให้แผนก IT สำหรับปัญหาทางเทคนิค ระบบ รหัสผ่าน VPN"""
    pass

@tool
def transfer_to_human():
    """โอนสายให้เจ้าหน้าที่ เมื่อไม่สามารถช่วยเหลือได้"""
    pass

# Triage Agent
def triage_agent(state: HandoffState):
    messages = [
        {"role": "system", "content": """คุณคือ Triage Agent ทำหน้าที่รับเรื่องแล้วส่งต่อ:
- คำถามเรื่องเงิน/ค่าใช้จ่าย → ใช้ transfer_to_billing
- คำถามเรื่อง IT/ระบบ → ใช้ transfer_to_tech
- แก้ไขไม่ได้ → ใช้ transfer_to_human
- คำถามทั่วไป → ตอบเอง"""},
        *state["messages"]
    ]
    model_with_tools = supervisor_model.bind_tools([transfer_to_billing, transfer_to_tech, transfer_to_human])
    response = model_with_tools.invoke(messages)

    # ตรวจว่า agent ต้องการ handoff ไหม
    if response.tool_calls:
        tool_name = response.tool_calls[0]["name"]
        if "billing" in tool_name:
            return {"current_agent": "billing", "messages": [response]}
        elif "tech" in tool_name:
            return {"current_agent": "tech", "messages": [response]}
        elif "human" in tool_name:
            return {"current_agent": "human", "messages": [response]}

    return {"current_agent": "done", "messages": [response]}

# Specialized Agents
def billing_agent(state: HandoffState):
    messages = [
        {"role": "system", "content": "คุณคือผู้เชี่ยวชาญด้านการเงิน ตอบเรื่องค่าใช้จ่าย การเบิกเงิน ใบเสร็จ"},
        *state["messages"]
    ]
    response = worker_model.invoke(messages)
    return {"messages": [response], "current_agent": "done"}

def tech_agent(state: HandoffState):
    messages = [
        {"role": "system", "content": "คุณคือ IT Support ตอบปัญหาเทคนิค ระบบ VPN รหัสผ่าน"},
        *state["messages"]
    ]
    response = worker_model.invoke(messages)
    return {"messages": [response], "current_agent": "done"}

def human_handoff(state: HandoffState):
    return {
        "messages": [{"role": "assistant", "content": "กำลังโอนสายให้เจ้าหน้าที่ กรุณารอสักครู่ค่ะ..."}],
        "current_agent": "done"
    }

# Routing
def route_triage(state: HandoffState) -> str:
    agent = state.get("current_agent", "done")
    return {"billing": "billing", "tech": "tech", "human": "human"}.get(agent, END)

# Build
builder = StateGraph(HandoffState)
builder.add_node("triage", triage_agent)
builder.add_node("billing", billing_agent)
builder.add_node("tech", tech_agent)
builder.add_node("human", human_handoff)

builder.add_edge(START, "triage")
builder.add_conditional_edges("triage", route_triage)
builder.add_edge("billing", END)
builder.add_edge("tech", END)
builder.add_edge("human", END)

handoff_system = builder.compile()

# ทดสอบ
tests = [
    "เบิกค่าเดินทางยังไงครับ",    # → billing
    "ลืมรหัสผ่าน VPN",            # → tech
    "อยากลาออก",                  # → human
]

for q in tests:
    r = handoff_system.invoke({"messages": [{"role": "user", "content": q}], "current_agent": ""})
    print(f"❓ {q}")
    print(f"💬 {r['messages'][-1].content[:100]}")
    print()
```

---

## 5. Swarm Pattern

### 5.1 แนวคิด

ทุก Agent สื่อสารกันได้โดยตรง ไม่มี hierarchy:

```python
# Swarm ใช้ LangGraph prebuilt
from langgraph.prebuilt import create_react_agent

# แต่ละ Agent มี handoff tools ไปยัง Agent อื่น
# Agent A สามารถ handoff ไป B, C
# Agent B สามารถ handoff ไป A, C
# Agent C สามารถ handoff ไป A, B
```

---

## 6. Pipeline Pattern

### 6.1 แนวคิด

Agent ทำงานเป็นสายพานต่อเนื่อง:

```
Input → [Translator] → [Summarizer] → [Formatter] → Output
```

### 6.2 Implementation

```python
class PipelineState(TypedDict):
    messages: Annotated[list, add_messages]
    original_text: str
    translated_text: str
    summary: str
    formatted_output: str

def translator(state: PipelineState):
    """แปลภาษา"""
    response = worker_model.invoke(
        f"แปลข้อความนี้เป็นภาษาอังกฤษ ตอบแค่คำแปล:\n{state['original_text']}"
    )
    print(f"🌐 Translator: done")
    return {"translated_text": response.content}

def summarizer(state: PipelineState):
    """สรุป"""
    response = worker_model.invoke(
        f"สรุปข้อความนี้เป็น 2 ประโยค:\n{state['translated_text']}"
    )
    print(f"📝 Summarizer: done")
    return {"summary": response.content}

def formatter(state: PipelineState):
    """จัดรูปแบบ"""
    output = f"""## Summary
{state['summary']}

## Original (Thai)
{state['original_text'][:200]}...

## Translation (English)
{state['translated_text'][:200]}..."""

    print(f"🎨 Formatter: done")
    return {"formatted_output": output}

# Build Pipeline
builder = StateGraph(PipelineState)
builder.add_node("translator", translator)
builder.add_node("summarizer", summarizer)
builder.add_node("formatter", formatter)

builder.add_edge(START, "translator")
builder.add_edge("translator", "summarizer")
builder.add_edge("summarizer", "formatter")
builder.add_edge("formatter", END)

pipeline = builder.compile()

result = pipeline.invoke({
    "messages": [],
    "original_text": "LangGraph เป็น library ที่สร้างบน LangChain สำหรับควบคุม Flow ของ Agent อย่างละเอียด ใช้แนวคิดของ Graph",
    "translated_text": "", "summary": "", "formatted_output": ""
})
print(result["formatted_output"])
```

---

## 7. Debate Pattern

### 7.1 แนวคิด

2 Agent ถกเถียงกัน แล้ว Judge ตัดสิน:

```python
class DebateState(TypedDict):
    messages: Annotated[list, add_messages]
    topic: str
    pro_argument: str
    con_argument: str
    judge_decision: str
    round: int

def pro_agent(state: DebateState):
    """Agent ฝ่ายสนับสนุน"""
    counter = state.get("con_argument", "")
    prompt = f"""คุณสนับสนุน: "{state['topic']}"
{"ฝ่ายค้านบอกว่า: " + counter if counter else "ให้เหตุผลสนับสนุน"}
ให้เหตุผลสนับสนุนอย่างมีน้ำหนัก ใน 2-3 ประโยค"""
    response = worker_model.invoke(prompt)
    return {"pro_argument": response.content, "round": state.get("round", 0) + 1}

def con_agent(state: DebateState):
    """Agent ฝ่ายค้าน"""
    prompt = f"""คุณคัดค้าน: "{state['topic']}"
ฝ่ายสนับสนุนบอกว่า: {state['pro_argument']}
ให้เหตุผลคัดค้านอย่างมีน้ำหนัก ใน 2-3 ประโยค"""
    response = worker_model.invoke(prompt)
    return {"con_argument": response.content}

def judge(state: DebateState):
    """Judge ตัดสิน"""
    response = worker_model.invoke(f"""ตัดสินการดีเบตนี้:
หัวข้อ: {state['topic']}
ฝ่ายสนับสนุน: {state['pro_argument']}
ฝ่ายค้าน: {state['con_argument']}

สรุปว่าฝ่ายไหนมีเหตุผลน้ำหนักกว่า และเพราะอะไร""")
    return {"judge_decision": response.content}

def should_continue(state: DebateState) -> Literal["con_agent", "judge"]:
    if state.get("round", 0) >= 2:  # debate 2 รอบแล้ว judge
        return "judge"
    return "con_agent"

builder = StateGraph(DebateState)
builder.add_node("pro_agent", pro_agent)
builder.add_node("con_agent", con_agent)
builder.add_node("judge", judge)

builder.add_edge(START, "pro_agent")
builder.add_conditional_edges("pro_agent", should_continue)
builder.add_edge("con_agent", "pro_agent")
builder.add_edge("judge", END)

debate = builder.compile()

result = debate.invoke({
    "messages": [], "topic": "AI จะมาแทนที่โปรแกรมเมอร์ได้หรือไม่",
    "pro_argument": "", "con_argument": "", "judge_decision": "", "round": 0
})

print(f"⚖️  Judge: {result['judge_decision']}")
```

---

## 9. ตัวอย่าง: Content Creation Team

ทีมสร้าง Content แบบครบวงจร:

```
User: "เขียน blog post เรื่อง LangGraph"
          │
          ▼
    [Supervisor] ──→ [SEO Analyst] → วิเคราะห์ keyword
          │
          ├──→ [Researcher] → ค้นหาข้อมูล
          │
          ├──→ [Writer] → เขียนบทความ
          │
          ├──→ [Editor] → ตรวจแก้ไข
          │
          └──→ [FINISH] → บทความพร้อมเผยแพร่
```

(ใช้ Supervisor Pattern จากหัวข้อ 3 โดยเพิ่ม Agents)

---

## 12. Error Handling

### 12.1 จำกัดจำนวนรอบ

```python
MAX_ITERATIONS = 10

def supervisor_with_limit(state: TeamState):
    iteration = state.get("iteration", 0)
    if iteration >= MAX_ITERATIONS:
        print(f"⚠️  ถึง limit {MAX_ITERATIONS} รอบแล้ว บังคับจบ")
        return {"next_agent": "FINISH"}

    # ... supervisor logic ...
    return {"next_agent": decision.next, "iteration": iteration + 1}
```

### 12.2 Timeout per Agent

```python
import asyncio

async def agent_with_timeout(state, timeout=30):
    try:
        result = await asyncio.wait_for(
            actual_agent_logic(state),
            timeout=timeout
        )
        return result
    except asyncio.TimeoutError:
        return {"messages": [{"role": "assistant", "content": "Agent timeout — ส่งต่อ Supervisor"}]}
```

### 12.3 Fallback Agent

```python
def agent_with_fallback(state: TeamState):
    try:
        return primary_agent(state)
    except Exception as e:
        print(f"⚠️  Primary agent failed: {e}")
        return fallback_agent(state)
```

---

## 13. Best Practices

```
✅ แต่ละ Agent ต้องมี role ชัดเจน
  - System prompt สั้น กระชับ เฉพาะ role นั้น
  - ไม่ทำหลาย role ใน Agent เดียว

✅ ใช้ Structured Output สำหรับการตัดสินใจ
  - Supervisor ควร return JSON/Pydantic ไม่ใช่ free text
  - ลดโอกาสที่ routing จะผิดพลาด

✅ จำกัด iterations เสมอ
  - ป้องกัน infinite loop
  - ตั้ง MAX_ITERATIONS ไว้ (10-20 สำหรับ complex tasks)

✅ Log ทุก handoff
  - ใช้ LangSmith trace ดูว่า Agent ไหนถูกเรียกบ้าง
  - วัด latency ของแต่ละ Agent

✅ ทดสอบแต่ละ Agent แยก
  - เขียน unit test สำหรับแต่ละ Agent
  - ทดสอบ routing logic แยกจาก Agent logic

✅ ใช้ model ที่เหมาะสม
  - Supervisor → model เก่ง reasoning (GPT-4o, Claude Sonnet)
  - Worker → model เร็ว ถูกกว่า (GPT-4o-mini, Gemini Flash)
  - Judge → model แม่นยำ

❌ อย่าสร้าง Agent เยอะเกินไป
  - เริ่มจาก 2-3 Agent
  - เพิ่มเมื่อจำเป็นจริงๆ
  - ยิ่งเยอะ ยิ่ง debug ยาก + latency สูง
```
