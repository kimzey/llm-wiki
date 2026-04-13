# LangGraph Deep Dive — ควบคุม Agent Flow อย่างละเอียด

> LangGraph คือ library สำหรับสร้าง stateful, multi-actor applications ด้วย LLM
> ใช้แนวคิดของ Graph (Nodes + Edges) เพื่อควบคุม workflow ที่ซับซ้อน

---

## สารบัญ

1. [Core Concepts — แนวคิดพื้นฐาน](#1-core-concepts)
2. [State — หัวใจของ LangGraph](#2-state)
3. [Nodes — ขั้นตอนการทำงาน](#3-nodes)
4. [Edges — เส้นทางการเชื่อมต่อ](#4-edges)
5. [Prebuilt Components — ของสำเร็จรูป](#5-prebuilt-components)
6. [Checkpointing — บันทึก State](#6-checkpointing)
7. [Human-in-the-Loop — คนอนุมัติ](#7-human-in-the-loop)
8. [Subgraphs — Graph ซ้อน Graph](#8-subgraphs)
9. [Multi-Agent Systems](#9-multi-agent-systems)
10. [Streaming — แสดงผลแบบ Real-time](#10-streaming)
11. [ตัวอย่าง: Customer Support Bot สมบูรณ์](#11-ตัวอย่าง-customer-support-bot)
12. [ตัวอย่าง: Research Agent](#12-ตัวอย่าง-research-agent)
13. [Deployment](#13-deployment)

---

## 1. Core Concepts

### LangGraph vs LangChain Agent

```
LangChain create_agent:
  User → [LLM ↔ Tools loop] → Answer
  (ควบคุมน้อย, ใช้ง่าย, เหมาะงานตรงไปตรงมา)

LangGraph:
  User → [Node A] → (เงื่อนไข?) → [Node B] → [Node C] → Answer
                                  → [Node D] → [Node B] → ...
  (ควบคุมทุกจุด, ยืดหยุ่นสูง, เหมาะงานซับซ้อน)
```

### องค์ประกอบหลัก

```python
# 1. State  = ข้อมูลที่ไหลผ่าน Graph (เหมือน "กระดาษจดบันทึก" ที่ทุก Node อ่าน/เขียนได้)
# 2. Node   = ฟังก์ชันที่ทำงานบางอย่าง (เช่น เรียก LLM, เรียก API, แปลงข้อมูล)
# 3. Edge   = เส้นเชื่อมระหว่าง Node (บอกว่าไป Node ไหนต่อ)
# 4. Graph  = โครงสร้างรวมที่ compile แล้วพร้อมรัน
```

### ติดตั้ง

```bash
pip install -U langgraph "langchain[google-genai]"
```

---

## 2. State

### 2.1 State คืออะไร ?

State คือ TypedDict ที่กำหนดโครงสร้างข้อมูลที่จะไหลผ่านทุก Node ใน Graph ทุก Node จะรับ State เข้ามา ทำงาน แล้ว return State ที่อัปเดตกลับไป

### 2.2 State พื้นฐาน — Messages Only

```python
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]
    # Annotated[list, add_messages] หมายถึง:
    # - เก็บเป็น list
    # - ใช้ add_messages เป็น "reducer" (วิธีรวมข้อมูลใหม่เข้ากับเก่า)
    # - add_messages จะ APPEND message ใหม่เข้าไป ไม่ใช่ REPLACE
```

### 2.3 Reducer คืออะไร ?

Reducer กำหนดว่าเมื่อ Node return ค่าใหม่ จะรวมกับค่าเก่าอย่างไร:

```python
from typing import Annotated
from operator import add

class State(TypedDict):
    # ไม่มี reducer → REPLACE (ค่าใหม่ทับค่าเก่า)
    current_step: str

    # add_messages reducer → APPEND messages
    messages: Annotated[list, add_messages]

    # operator.add reducer → รวม list เข้าด้วยกัน
    collected_data: Annotated[list, add]

    # Custom reducer
    counter: Annotated[int, lambda old, new: old + new]
```

**ตัวอย่างการทำงานของ Reducer:**

```python
# สมมติ State ปัจจุบัน:
# {
#   "current_step": "step_1",
#   "messages": [msg1, msg2],
#   "collected_data": ["a", "b"],
#   "counter": 5
# }

# Node return:
# {
#   "current_step": "step_2",        → REPLACE: "step_2"
#   "messages": [msg3],              → APPEND: [msg1, msg2, msg3]
#   "collected_data": ["c"],         → ADD: ["a", "b", "c"]
#   "counter": 3                     → CUSTOM: 5 + 3 = 8
# }
```

### 2.4 State ซับซ้อน — เก็บหลายข้อมูล

```python
from typing import Annotated, Optional
from typing_extensions import TypedDict
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    # ข้อความบทสนทนา
    messages: Annotated[list, add_messages]

    # ข้อมูลที่เก็บระหว่างทาง
    user_intent: str              # เจตนาของผู้ใช้
    category: str                 # หมวดหมู่คำถาม
    retrieved_docs: list[str]     # เอกสารที่ดึงมา
    confidence: float             # ความมั่นใจในคำตอบ
    needs_human_review: bool      # ต้องให้คนตรวจไหม
    iteration_count: int          # นับรอบ
```

### 2.5 Input/Output Schema แยกกัน

```python
from langgraph.graph import StateGraph

class InputState(TypedDict):
    question: str

class OutputState(TypedDict):
    answer: str
    sources: list[str]

class FullState(InputState, OutputState):
    messages: Annotated[list, add_messages]
    intermediate_steps: list[str]

# Graph ที่รับ Input แบบหนึ่ง และ Output อีกแบบหนึ่ง
graph = StateGraph(FullState, input=InputState, output=OutputState)
```

---

## 3. Nodes

### 3.1 Node คืออะไร ?

Node คือ function ที่:
- **รับ State** เข้ามา
- **ทำงาน** บางอย่าง (เรียก LLM, เรียก API, คำนวณ, etc.)
- **return dict** ที่จะอัปเดต State

```python
# Node ง่ายๆ
def greet(state: State) -> dict:
    return {"messages": [{"role": "assistant", "content": "สวัสดีครับ!"}]}

# Node ที่เรียก LLM
def call_llm(state: State) -> dict:
    response = model.invoke(state["messages"])
    return {"messages": [response]}

# Node ที่ทำ Logic
def check_confidence(state: State) -> dict:
    if state["confidence"] < 0.5:
        return {"needs_human_review": True}
    return {"needs_human_review": False}
```

### 3.2 เพิ่ม Node เข้า Graph

```python
from langgraph.graph import StateGraph, START, END

graph_builder = StateGraph(State)

# เพิ่ม Node
graph_builder.add_node("greet", greet)
graph_builder.add_node("call_llm", call_llm)
graph_builder.add_node("check", check_confidence)

# START และ END เป็น virtual nodes พิเศษ
# START = จุดเริ่มต้น (ไม่ต้อง add)
# END = จุดสิ้นสุด (ไม่ต้อง add)
```

### 3.3 Node ที่เป็น Class

```python
class RAGNode:
    def __init__(self, retriever, model):
        self.retriever = retriever
        self.model = model

    def __call__(self, state: State) -> dict:
        question = state["messages"][-1].content
        docs = self.retriever.invoke(question)
        context = "\n".join([d.page_content for d in docs])

        prompt = f"ตอบจากข้อมูลนี้:\n{context}\n\nคำถาม: {question}"
        response = self.model.invoke(prompt)

        return {
            "messages": [response],
            "retrieved_docs": [d.page_content for d in docs],
        }

# ใช้งาน
rag_node = RAGNode(retriever=my_retriever, model=model)
graph_builder.add_node("rag", rag_node)
```

---

## 4. Edges

### 4.1 Normal Edge — เส้นตรง

```python
# จาก A ไป B เสมอ (ไม่มีเงื่อนไข)
graph_builder.add_edge(START, "greet")
graph_builder.add_edge("greet", "call_llm")
graph_builder.add_edge("call_llm", END)

# Flow: START → greet → call_llm → END
```

### 4.2 Conditional Edge — เส้นมีเงื่อนไข

```python
from typing import Literal

def route_by_category(state: State) -> Literal["technical", "general", "complaint"]:
    """ฟังก์ชัน routing ที่ return ชื่อ Node ถัดไป"""
    category = state["category"]
    if category == "technical":
        return "technical"
    elif category == "complaint":
        return "complaint"
    else:
        return "general"

# เพิ่ม conditional edge
graph_builder.add_conditional_edges(
    "classifier",          # จาก Node นี้
    route_by_category,     # ใช้ฟังก์ชันนี้ตัดสินใจ
    # (optional) mapping ชื่อ output → ชื่อ Node
    # {
    #     "technical": "tech_support",
    #     "general": "general_support",
    #     "complaint": "complaint_handler",
    # }
)
```

### 4.3 tools_condition — Built-in สำหรับ Tool Calling

```python
from langgraph.prebuilt import tools_condition

# tools_condition ตรวจสอบอัตโนมัติ:
# - ถ้า AI message มี tool_calls → ไปที่ "tools"
# - ถ้าไม่มี → ไปที่ END

graph_builder.add_conditional_edges("agent", tools_condition)
# เทียบเท่ากับ:
# def should_call_tool(state):
#     last_msg = state["messages"][-1]
#     if last_msg.tool_calls:
#         return "tools"
#     return END
```

### 4.4 ตัวอย่าง Graph แบบต่างๆ

```python
# === Linear (เส้นตรง) ===
# START → A → B → C → END

# === Branching (แยกทาง) ===
# START → Classify → (tech?) → TechBot → END
#                   → (general?) → GeneralBot → END

# === Loop (วน) ===
# START → Agent → (need tool?) → Tools → Agent → (done?) → END

# === Fan-out / Fan-in (แยกแล้วรวม) ===
# START → Splitter → [A, B, C] (parallel) → Merger → END

# === Nested (ซ้อนกัน) ===
# START → OuterNode → [SubGraph: X → Y → Z] → OuterNode2 → END
```

---

## 5. Prebuilt Components

### 5.1 ToolNode — Node สำหรับรัน Tools

```python
from langgraph.prebuilt import ToolNode

tools = [get_weather, calculate, search_database]

# ToolNode จะ:
# 1. อ่าน tool_calls จาก AI message ล่าสุด
# 2. รัน tools ที่ถูกเรียก
# 3. Return ผลลัพธ์เป็น ToolMessage

graph_builder.add_node("tools", ToolNode(tools=tools))
```

### 5.2 create_react_agent — Prebuilt ReAct Agent

ถ้าต้องการ Agent แบบง่ายๆ ไม่ต้องสร้าง Graph เอง:

```python
from langgraph.prebuilt import create_react_agent

# สร้าง Agent สำเร็จรูป (ภายในสร้าง Graph ให้แล้ว)
agent = create_react_agent(
    model=model,
    tools=[get_weather, calculate],
    state_modifier="คุณคือผู้ช่วย AI ที่ตอบเป็นภาษาไทย",  # system prompt
)

# ใช้งาน
result = agent.invoke({
    "messages": [{"role": "user", "content": "อากาศที่กรุงเทพเป็นยังไง?"}]
})
```

### 5.3 เปรียบเทียบ create_react_agent vs สร้างเอง

```python
# === Prebuilt (ง่าย, ควบคุมน้อย) ===
agent = create_react_agent(model, tools, state_modifier="...")
# ภายในสร้าง Graph:  agent_node ↔ tools (loop)

# === สร้างเอง (ยากขึ้น, ควบคุมเต็มที่) ===
graph = StateGraph(State)
graph.add_node("classifier", classify)
graph.add_node("agent", call_llm)
graph.add_node("tools", ToolNode(tools))
graph.add_node("reviewer", human_review)
graph.add_edge(START, "classifier")
graph.add_conditional_edges("classifier", route)
graph.add_conditional_edges("agent", tools_condition)
graph.add_edge("tools", "reviewer")
graph.add_edge("reviewer", "agent")
# ← ทำอะไรก็ได้ตามใจ
```

---

## 6. Checkpointing

### 6.1 ทำไมต้อง Checkpoint ?

- **Persistence:** เก็บ state ไว้ เพื่อกลับมาทำต่อได้ (เช่น user ปิด browser แล้วกลับมา)
- **Human-in-the-loop:** หยุดกลางทาง รอคนอนุมัติ แล้วทำต่อ
- **Time-travel:** ย้อนกลับไปดู state ณ จุดใดก็ได้
- **Multi-turn:** เก็บประวัติบทสนทนาข้าม invoke

### 6.2 MemorySaver (In-Memory)

```python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
graph = graph_builder.compile(checkpointer=memory)

# ต้องระบุ thread_id เพื่อแยก session
config = {"configurable": {"thread_id": "user-123"}}

# รอบที่ 1
result1 = graph.invoke(
    {"messages": [{"role": "user", "content": "ผมชื่อ สมชาย"}]},
    config
)

# รอบที่ 2 — AI จำได้ว่าชื่อ สมชาย (เพราะ checkpoint เก็บ state ไว้)
result2 = graph.invoke(
    {"messages": [{"role": "user", "content": "ผมชื่ออะไร?"}]},
    config
)
print(result2["messages"][-1].content)  # "คุณชื่อ สมชาย ครับ"

# thread ใหม่ → ไม่รู้จัก
config_new = {"configurable": {"thread_id": "user-456"}}
result3 = graph.invoke(
    {"messages": [{"role": "user", "content": "ผมชื่ออะไร?"}]},
    config_new
)
print(result3["messages"][-1].content)  # "ผมไม่ทราบชื่อของคุณครับ"
```

### 6.3 SqliteSaver (Persistent — เก็บลงไฟล์)

```bash
pip install langgraph-checkpoint-sqlite
```

```python
from langgraph.checkpoint.sqlite import SqliteSaver

# เก็บลงไฟล์ .db
with SqliteSaver.from_conn_string("checkpoints.db") as memory:
    graph = graph_builder.compile(checkpointer=memory)
    # ... ใช้งานปกติ
    # state จะถูกเก็บใน checkpoints.db
    # แม้ restart program state ก็ยังอยู่
```

### 6.4 PostgresSaver (Production)

```bash
pip install langgraph-checkpoint-postgres
```

```python
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://user:pass@localhost:5432/mydb"

with PostgresSaver.from_conn_string(DB_URI) as memory:
    memory.setup()  # สร้างตาราง (ทำครั้งแรก)
    graph = graph_builder.compile(checkpointer=memory)
```

### 6.5 ดู State History

```python
# ดู state ปัจจุบัน
current_state = graph.get_state(config)
print(current_state.values)      # dict ของ state
print(current_state.next)        # Node ถัดไป (ถ้ายังไม่จบ)
print(current_state.created_at)  # เวลาที่สร้าง

# ดู history ทั้งหมด
for state in graph.get_state_history(config):
    print(f"Step: {state.metadata.get('step', '?')}")
    print(f"  Next: {state.next}")
    print(f"  Messages: {len(state.values.get('messages', []))}")
```

---

## 7. Human-in-the-Loop

### 7.1 interrupt_before — หยุดก่อน Node

```python
graph = graph_builder.compile(
    checkpointer=memory,
    interrupt_before=["tools"],  # หยุดก่อนเรียก tools ทุกครั้ง
)

config = {"configurable": {"thread_id": "review-1"}}

# Step 1: Agent ตัดสินใจจะเรียก tool แต่ถูกหยุดก่อน
result = graph.invoke(
    {"messages": [{"role": "user", "content": "ลบข้อมูลลูกค้า ID 12345"}]},
    config
)

# Step 2: ดูว่า agent ต้องการทำอะไร
state = graph.get_state(config)
pending_tool = result["messages"][-1].tool_calls
print(f"Agent ต้องการเรียก: {pending_tool}")
# [{'name': 'delete_customer', 'args': {'id': '12345'}}]

# Step 3: คนตัดสินใจ
approval = input("อนุมัติ? (y/n): ")

if approval == "y":
    # ทำต่อจากจุดที่หยุด
    result = graph.invoke(None, config)
    print(result["messages"][-1].content)
else:
    # ยกเลิก — เพิ่ม message บอก AI ว่าถูกปฏิเสธ
    from langchain_core.messages import ToolMessage

    tool_call_id = pending_tool[0]["id"]
    graph.update_state(
        config,
        {
            "messages": [
                ToolMessage(
                    content="ผู้ดูแลปฏิเสธการดำเนินการนี้",
                    tool_call_id=tool_call_id,
                )
            ]
        },
    )
    result = graph.invoke(None, config)
    print(result["messages"][-1].content)
    # "ขอโทษครับ การลบข้อมูลถูกปฏิเสธโดยผู้ดูแลระบบ"
```

### 7.2 interrupt_after — หยุดหลัง Node

```python
graph = graph_builder.compile(
    checkpointer=memory,
    interrupt_after=["draft_email"],  # หยุดหลังร่างอีเมลเสร็จ
)

# Flow: user → draft_email (หยุด) → คนตรวจ → send_email
```

### 7.3 Dynamic Interrupt (ใน Node)

```python
from langgraph.types import interrupt

def sensitive_action(state: State) -> dict:
    action = state.get("pending_action", "")

    # ตรวจสอบว่าต้องขออนุมัติไหม
    if "delete" in action or "transfer" in action:
        # หยุดและถามคน
        approval = interrupt(
            value=f"ต้องการอนุมัติ: {action}",
        )
        if approval != "approved":
            return {"messages": [{"role": "assistant", "content": "ยกเลิกการดำเนินการ"}]}

    # ดำเนินการ
    return {"messages": [{"role": "assistant", "content": f"ดำเนินการ {action} เสร็จสิ้น"}]}
```

---

## 8. Subgraphs

### 8.1 Graph ซ้อน Graph

```python
from langgraph.graph import StateGraph, START, END

# === Inner Graph: RAG Pipeline ===
class RAGState(TypedDict):
    messages: Annotated[list, add_messages]
    context: str

def retrieve(state: RAGState):
    # ... ดึงข้อมูลจาก Vector Store
    return {"context": "ข้อมูลที่ดึงมา..."}

def generate(state: RAGState):
    # ... สร้างคำตอบจาก context
    response = model.invoke(f"Context: {state['context']}\n\nQ: {state['messages'][-1].content}")
    return {"messages": [response]}

rag_builder = StateGraph(RAGState)
rag_builder.add_node("retrieve", retrieve)
rag_builder.add_node("generate", generate)
rag_builder.add_edge(START, "retrieve")
rag_builder.add_edge("retrieve", "generate")
rag_builder.add_edge("generate", END)
rag_graph = rag_builder.compile()

# === Outer Graph: Main Agent ===
class MainState(TypedDict):
    messages: Annotated[list, add_messages]
    context: str
    category: str

def classify(state: MainState):
    # ... จำแนกหมวดหมู่
    return {"category": "knowledge_base"}

outer_builder = StateGraph(MainState)
outer_builder.add_node("classify", classify)
outer_builder.add_node("rag", rag_graph)  # ← ใส่ graph เป็น Node!
outer_builder.add_node("general", call_llm)

outer_builder.add_edge(START, "classify")
outer_builder.add_conditional_edges("classify", lambda s: s["category"])
outer_builder.add_edge("rag", END)
outer_builder.add_edge("general", END)

main_graph = outer_builder.compile()
```

---

## 9. Multi-Agent Systems

### 9.1 หลายๆ Agent ทำงานร่วมกัน

```python
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain.tools import tool
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from typing import Annotated, Literal
from typing_extensions import TypedDict

model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

class MultiAgentState(TypedDict):
    messages: Annotated[list, add_messages]
    current_agent: str
    task_type: str

# --- Agent 1: Researcher ---
def researcher(state: MultiAgentState):
    messages = [
        {"role": "system", "content": """คุณคือนักวิจัย ทำหน้าที่ค้นหาและรวบรวมข้อมูล
ตอบกลับด้วยข้อมูลที่ค้นพบพร้อมแหล่งที่มา"""},
        *state["messages"]
    ]
    response = model.invoke(messages)
    return {"messages": [response], "current_agent": "researcher"}

# --- Agent 2: Writer ---
def writer(state: MultiAgentState):
    messages = [
        {"role": "system", "content": """คุณคือนักเขียน ทำหน้าที่เรียบเรียงข้อมูลที่ได้รับ
ให้เป็นบทความที่อ่านง่ายและน่าสนใจ"""},
        *state["messages"]
    ]
    response = model.invoke(messages)
    return {"messages": [response], "current_agent": "writer"}

# --- Agent 3: Reviewer ---
def reviewer(state: MultiAgentState):
    messages = [
        {"role": "system", "content": """คุณคือบรรณาธิการ ทำหน้าที่ตรวจสอบบทความ
ให้ feedback ว่าต้องแก้ไขอะไร หรือถ้า OK ให้ตอบว่า "APPROVED" """},
        *state["messages"]
    ]
    response = model.invoke(messages)
    return {"messages": [response], "current_agent": "reviewer"}

# --- Routing ---
def route_after_review(state: MultiAgentState) -> Literal["writer", END]:
    last_msg = state["messages"][-1].content
    if "APPROVED" in last_msg.upper():
        return END
    return "writer"  # ส่งกลับไปแก้

# --- สร้าง Graph ---
builder = StateGraph(MultiAgentState)

builder.add_node("researcher", researcher)
builder.add_node("writer", writer)
builder.add_node("reviewer", reviewer)

builder.add_edge(START, "researcher")
builder.add_edge("researcher", "writer")
builder.add_edge("writer", "reviewer")
builder.add_conditional_edges("reviewer", route_after_review)

multi_agent = builder.compile()

# --- ใช้งาน ---
result = multi_agent.invoke({
    "messages": [{"role": "user", "content": "เขียนบทความเรื่อง LangGraph"}],
    "current_agent": "",
    "task_type": "article",
})

# ดู flow ที่เกิดขึ้น
for msg in result["messages"]:
    print(f"[{msg.type}] {msg.content[:80]}...")
    print()
```

**Flow:**
```
User: "เขียนบทความเรื่อง LangGraph"
  → Researcher: ค้นหาข้อมูลเกี่ยวกับ LangGraph...
  → Writer: เรียบเรียงเป็นบทความ...
  → Reviewer: "ต้องเพิ่มตัวอย่าง code"
  → Writer: (แก้ไข) เพิ่มตัวอย่าง code...
  → Reviewer: "APPROVED"
  → END
```

---

## 10. Streaming

### 10.1 Stream Events ทีละ Node

```python
# stream() จะ yield ผลลัพธ์ของแต่ละ Node ทีละตัว
for event in graph.stream(
    {"messages": [{"role": "user", "content": "สวัสดี"}]},
    config
):
    # event = {node_name: output_dict}
    for node_name, output in event.items():
        print(f"--- Node: {node_name} ---")
        if "messages" in output:
            print(output["messages"][-1].content[:100])
```

### 10.2 Stream Mode: values

```python
# stream_mode="values" → yield state ทั้งหมดหลังแต่ละ step
for state in graph.stream(
    {"messages": [{"role": "user", "content": "สวัสดี"}]},
    config,
    stream_mode="values"
):
    print(f"Messages count: {len(state['messages'])}")
    print(f"Latest: {state['messages'][-1].content[:50]}")
    print("---")
```

### 10.3 Stream Tokens (ทีละ token)

```python
# astream_events() → stream ทุก event รวมถึง LLM tokens
async for event in graph.astream_events(
    {"messages": [{"role": "user", "content": "เล่านิทาน"}]},
    config,
    version="v2"
):
    if event["event"] == "on_chat_model_stream":
        token = event["data"]["chunk"].content
        print(token, end="", flush=True)
```

---

## 11. ตัวอย่าง: Customer Support Bot สมบูรณ์

```python
import os
from dotenv import load_dotenv
from typing import Annotated, Literal
from typing_extensions import TypedDict

from langchain_google_genai import ChatGoogleGenerativeAI
from langchain.tools import tool
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.checkpoint.memory import MemorySaver

load_dotenv()

# ======== State ========
class SupportState(TypedDict):
    messages: Annotated[list, add_messages]
    category: str           # billing, technical, general
    sentiment: str          # positive, negative, neutral
    escalated: bool

# ======== Tools ========
@tool
def lookup_order(order_id: str) -> str:
    """ค้นหาข้อมูลคำสั่งซื้อจาก Order ID"""
    orders = {
        "ORD-001": "สถานะ: กำลังจัดส่ง, ETA: 3 วัน, สินค้า: iPhone 15",
        "ORD-002": "สถานะ: จัดส่งแล้ว, วันที่ส่ง: 1 มี.ค. 2025",
        "ORD-003": "สถานะ: ยกเลิก, เหตุผล: ลูกค้าขอยกเลิก",
    }
    return orders.get(order_id, f"ไม่พบ order {order_id}")

@tool
def check_warranty(product_name: str) -> str:
    """ตรวจสอบประกันสินค้า"""
    return f"{product_name}: ประกัน 1 ปี, เหลืออีก 8 เดือน, ครอบคลุมซ่อมฟรี"

@tool
def create_ticket(subject: str, priority: str) -> str:
    """สร้าง support ticket"""
    return f"สร้าง Ticket #TK-{hash(subject) % 10000:04d} เรื่อง '{subject}' ระดับ {priority} เรียบร้อย"

tools = [lookup_order, check_warranty, create_ticket]

# ======== Model ========
model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)
model_with_tools = model.bind_tools(tools)

# ======== Nodes ========
def classify_intent(state: SupportState):
    """จำแนกหมวดหมู่และ sentiment"""
    prompt = [
        {"role": "system", "content": """วิเคราะห์ข้อความของลูกค้า:
1. category: billing, technical, general
2. sentiment: positive, negative, neutral
ตอบในรูปแบบ: category|sentiment"""},
        state["messages"][-1]
    ]
    response = model.invoke(prompt)
    parts = response.content.strip().split("|")
    category = parts[0].strip() if len(parts) > 0 else "general"
    sentiment = parts[1].strip() if len(parts) > 1 else "neutral"

    return {"category": category, "sentiment": sentiment, "escalated": False}

def support_agent(state: SupportState):
    """Agent หลักที่ตอบลูกค้า"""
    system = f"""คุณคือ Customer Support ของบริษัท ABC
หมวดหมู่คำถาม: {state.get('category', 'general')}
อารมณ์ลูกค้า: {state.get('sentiment', 'neutral')}

กฎ:
- ถ้าลูกค้าไม่พอใจ (negative) ต้องขอโทษก่อนเสมอ
- ใช้เครื่องมือค้นหาข้อมูลก่อนตอบ
- ถ้าแก้ไม่ได้ ให้สร้าง ticket"""

    messages = [{"role": "system", "content": system}] + state["messages"]
    response = model_with_tools.invoke(messages)
    return {"messages": [response]}

def check_escalation(state: SupportState):
    """ตรวจสอบว่าต้อง escalate ไหม"""
    if state.get("sentiment") == "negative" and len(state["messages"]) > 6:
        return {"escalated": True}
    return {"escalated": False}

def escalate_to_human(state: SupportState):
    """ส่งต่อให้มนุษย์"""
    return {
        "messages": [{
            "role": "assistant",
            "content": "ขออภัยที่ยังไม่สามารถแก้ไขปัญหาได้ครับ ผมขอส่งต่อให้เจ้าหน้าที่ดูแลโดยตรงนะครับ กรุณารอสักครู่..."
        }]
    }

# ======== Routing ========
def route_after_check(state: SupportState) -> Literal["escalate", "agent"]:
    if state.get("escalated"):
        return "escalate"
    return "agent"

# ======== Build Graph ========
builder = StateGraph(SupportState)

builder.add_node("classify", classify_intent)
builder.add_node("check_escalation", check_escalation)
builder.add_node("agent", support_agent)
builder.add_node("tools", ToolNode(tools=tools))
builder.add_node("escalate", escalate_to_human)

# Edges
builder.add_edge(START, "classify")
builder.add_edge("classify", "check_escalation")
builder.add_conditional_edges("check_escalation", route_after_check)
builder.add_conditional_edges("agent", tools_condition)
builder.add_edge("tools", "agent")
builder.add_edge("escalate", END)

# Compile
memory = MemorySaver()
support_bot = builder.compile(checkpointer=memory)

# ======== ทดสอบ ========
config = {"configurable": {"thread_id": "customer-001"}}

# รอบ 1
r1 = support_bot.invoke(
    {"messages": [{"role": "user", "content": "สถานะ order ORD-001 เป็นยังไงบ้างครับ"}],
     "category": "", "sentiment": "", "escalated": False},
    config
)
print("Bot:", r1["messages"][-1].content)

# รอบ 2 (ต่อบทสนทนา)
r2 = support_bot.invoke(
    {"messages": [{"role": "user", "content": "มีประกันไหมครับ?"}]},
    config
)
print("Bot:", r2["messages"][-1].content)
```

---

## 12. ตัวอย่าง: Research Agent

Agent ที่วิจัยหัวข้อ โดยแบ่งเป็น Plan → Research → Write → Review:

```python
class ResearchState(TypedDict):
    messages: Annotated[list, add_messages]
    topic: str
    outline: list[str]
    research_notes: Annotated[list, lambda x, y: x + y]
    draft: str
    feedback: str
    revision_count: int

def planner(state: ResearchState):
    """วางแผนหัวข้อย่อย"""
    response = model.invoke(
        f"วางแผนบทความเรื่อง '{state['topic']}' ให้เป็น 3-5 หัวข้อย่อย ตอบเป็น list"
    )
    outline = response.content.strip().split("\n")
    return {"outline": outline, "revision_count": 0}

def researcher(state: ResearchState):
    """ค้นหาข้อมูลแต่ละหัวข้อ"""
    notes = []
    for topic in state["outline"][:3]:  # จำกัด 3 หัวข้อ
        response = model.invoke(f"ค้นหาข้อมูลสำคัญเกี่ยวกับ: {topic}")
        notes.append(f"## {topic}\n{response.content}")
    return {"research_notes": notes}

def writer(state: ResearchState):
    """เขียนบทความ"""
    notes = "\n\n".join(state["research_notes"])
    feedback = state.get("feedback", "")

    prompt = f"""เขียนบทความเรื่อง '{state['topic']}'
จาก notes นี้:
{notes}

{"Feedback ที่ต้องแก้ไข: " + feedback if feedback else ""}"""

    response = model.invoke(prompt)
    return {"draft": response.content}

def reviewer(state: ResearchState):
    """ตรวจสอบบทความ"""
    response = model.invoke(
        f"ตรวจสอบบทความนี้ ถ้าดีแล้วตอบ 'APPROVED' ถ้าต้องแก้ให้บอก feedback:\n\n{state['draft']}"
    )
    return {
        "feedback": response.content,
        "revision_count": 1  # จะถูก replace (ไม่มี reducer)
    }

def route_review(state: ResearchState) -> Literal["writer", END]:
    if "APPROVED" in state["feedback"].upper():
        return END
    if state.get("revision_count", 0) >= 2:  # จำกัด 2 รอบ
        return END
    return "writer"

# Build
builder = StateGraph(ResearchState)
builder.add_node("planner", planner)
builder.add_node("researcher", researcher)
builder.add_node("writer", writer)
builder.add_node("reviewer", reviewer)

builder.add_edge(START, "planner")
builder.add_edge("planner", "researcher")
builder.add_edge("researcher", "writer")
builder.add_edge("writer", "reviewer")
builder.add_conditional_edges("reviewer", route_review)

research_agent = builder.compile()

result = research_agent.invoke({
    "messages": [],
    "topic": "LangGraph คืออะไร และใช้งานอย่างไร",
    "outline": [],
    "research_notes": [],
    "draft": "",
    "feedback": "",
    "revision_count": 0,
})

print(result["draft"])
```

---

## 13. Deployment

### 13.1 LangGraph API Server

```bash
pip install langgraph-api
```

```python
# server.py
from langgraph.graph import StateGraph
# ... สร้าง graph ...

# Export graph
graph = builder.compile(checkpointer=memory)
```

```yaml
# langgraph.json
{
  "graphs": {
    "support_bot": "./server.py:graph"
  }
}
```

```bash
langgraph up
# → API server at http://localhost:8123
```

### 13.2 ใช้ผ่าน LangGraph SDK

```python
from langgraph_sdk import get_client

client = get_client(url="http://localhost:8123")

# สร้าง thread
thread = await client.threads.create()

# ส่งข้อความ
result = await client.runs.create(
    thread["thread_id"],
    "support_bot",
    input={"messages": [{"role": "user", "content": "สวัสดีครับ"}]}
)
```

### 13.3 Deploy บน LangGraph Cloud

```bash
# ใช้ LangSmith UI หรือ CLI
langgraph deploy --project my-agent
```

---

## Quick Reference

| ต้องการ | ใช้ |
|---------|-----|
| กำหนดข้อมูลที่ไหลผ่าน Graph | `State (TypedDict)` |
| สร้างขั้นตอนทำงาน | `graph.add_node("name", func)` |
| เชื่อม A → B ตรงๆ | `graph.add_edge("A", "B")` |
| เชื่อมแบบมีเงื่อนไข | `graph.add_conditional_edges("A", router)` |
| Auto route tool calls | `tools_condition` |
| รัน tools อัตโนมัติ | `ToolNode(tools=[...])` |
| เก็บ state ข้าม invoke | `compile(checkpointer=MemorySaver())` |
| หยุดให้คนอนุมัติ | `compile(interrupt_before=["node"])` |
| Agent สำเร็จรูป | `create_react_agent(model, tools)` |
| Stream ทีละ step | `graph.stream(input, config)` |
| Graph ซ้อน Graph | `graph.add_node("sub", sub_graph)` |
