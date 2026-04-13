# Step 4: LangGraph — ควบคุม Flow ด้วย Graph
## สร้าง Agent ที่มี workflow ซับซ้อน ตั้งแต่พื้นฐาน

> Step นี้ทำให้ Agent ทำงานได้ซับซ้อนขึ้น: แยกทาง, วน loop, ให้คนอนุมัติ, จำบทสนทนา

---

## 4.1 ติดตั้ง

```bash
pip install langgraph
```

## 4.2 LangGraph คืออะไร (อธิบายง่ายๆ)

```
LangChain Agent (create_agent):
  เหมือน "พนักงาน 1 คน" ที่ทำทุกอย่างเอง
  → ดีสำหรับงานง่ายๆ แต่ควบคุมยาก

LangGraph:
  เหมือน "แผนผังการทำงาน" ที่กำหนดว่า
  ใครทำอะไร → ส่งต่อให้ใคร → ตรวจสอบอะไร → จบตอนไหน
  → ควบคุมได้ทุกจุด
```

### องค์ประกอบ 4 อย่าง

```python
# 1. State  = "กระดาษจดบันทึก" ที่ทุกขั้นตอนอ่าน/เขียนได้
# 2. Node   = "ขั้นตอน" (เช่น ถาม AI, ค้นหาข้อมูล, ตรวจสอบ)
# 3. Edge   = "ลูกศร" เชื่อมขั้นตอน (A → B)
# 4. Graph  = "แผนผัง" ที่รวมทุกอย่างเข้าด้วยกัน
```

---

## 4.3 Graph แรก — แบบง่ายที่สุด

```python
# file: 22_first_graph.py

from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langchain_google_genai import ChatGoogleGenerativeAI
from dotenv import load_dotenv

load_dotenv()

# ==== 1. กำหนด State ====
class State(TypedDict):
    messages: Annotated[list, add_messages]
    # Annotated[list, add_messages] อธิบาย:
    #
    # list              = เก็บเป็น list
    # add_messages      = "reducer" (วิธีรวมข้อมูล)
    #
    # reducer คืออะไร?
    # = กฎว่าเมื่อ Node return ข้อมูลใหม่ จะรวมกับข้อมูลเก่าอย่างไร
    #
    # add_messages = APPEND (เพิ่มต่อท้าย)
    # ถ้าไม่มี reducer = REPLACE (ทับเลย)
    #
    # ตัวอย่าง:
    # State เดิม: messages = [msg1, msg2]
    # Node return: {"messages": [msg3]}
    # ผลลัพธ์:    messages = [msg1, msg2, msg3]  ← APPEND

# ==== 2. สร้าง Model ====
model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

# ==== 3. สร้าง Node (function) ====
def chatbot(state: State):
    """Node: ส่ง messages ให้ AI แล้ว return คำตอบ"""
    response = model.invoke(state["messages"])
    #                       ▲
    #                  อ่าน messages จาก State
    return {"messages": [response]}
    #       ▲
    #  return dict → ค่าจะถูกอัปเดตเข้า State
    #  messages: [response] → จะถูก APPEND เข้า list เดิม

# ==== 4. สร้าง Graph ====
builder = StateGraph(State)
#         ▲
#    StateGraph(State) = สร้าง graph ที่ใช้ State schema นี้

# เพิ่ม Node
builder.add_node("chatbot", chatbot)
#                 ▲          ▲
#            ชื่อ Node    function

# เพิ่ม Edges
builder.add_edge(START, "chatbot")  # START → chatbot
builder.add_edge("chatbot", END)    # chatbot → END
#                 ▲           ▲
#            จาก Node     ไป Node
#
# START = จุดเริ่มต้น (virtual node)
# END   = จุดจบ (virtual node)

# Compile
graph = builder.compile()
#        ▲
#   compile() = แปลง builder เป็น graph ที่รันได้

# ==== 5. ใช้งาน ====
result = graph.invoke({
    "messages": [{"role": "user", "content": "สวัสดีครับ"}]
})

for msg in result["messages"]:
    print(f"[{msg.type}] {msg.content}")
```

**Output:**
```
[human] สวัสดีครับ
[ai] สวัสดีครับ! มีอะไรให้ช่วยไหมครับ?
```

**Flow:**
```
START → chatbot → END
```

---

## 4.4 เพิ่ม Tools — Agent Loop

```python
# file: 23_graph_with_tools.py

from langchain.tools import tool
from langgraph.prebuilt import ToolNode, tools_condition

# ---- Tools ----
@tool
def get_weather(city: str) -> str:
    """ดูสภาพอากาศปัจจุบัน"""
    data = {"กรุงเทพ": "35°C แดดจัด", "เชียงใหม่": "28°C เย็นสบาย"}
    return data.get(city, f"ไม่มีข้อมูลของ {city}")

tools = [get_weather]

# ---- Model + bind tools ----
model_with_tools = model.bind_tools(tools)

# ---- Node ----
def agent(state: State):
    response = model_with_tools.invoke(state["messages"])
    return {"messages": [response]}

# ---- Build Graph ----
builder = StateGraph(State)

builder.add_node("agent", agent)
builder.add_node("tools", ToolNode(tools=tools))
#                          ▲
#                ToolNode คืออะไร?
#                = Node สำเร็จรูปที่:
#                  1. อ่าน tool_calls จาก AIMessage ล่าสุด
#                  2. รัน tools ที่ถูกเรียก
#                  3. Return ผลเป็น ToolMessage

# ---- Edges ----
builder.add_edge(START, "agent")

builder.add_conditional_edges("agent", tools_condition)
#       ▲                               ▲
#  add_conditional_edges คืออะไร?      tools_condition คืออะไร?
#  = เส้นเชื่อมที่ไปคนละทาง            = function สำเร็จรูปที่:
#    ขึ้นอยู่กับเงื่อนไข                  ถ้า AI มี tool_calls → ไปที่ "tools"
#                                        ถ้าไม่มี → ไปที่ END

builder.add_edge("tools", "agent")
#  หลัง tools ทำเสร็จ → กลับไปที่ agent
#  agent จะดูผลลัพธ์แล้วตัดสินใจ:
#    - เรียก tool อีก? → ไป tools อีกรอบ
#    - พอแล้ว?        → ไป END

graph = builder.compile()

# ---- ทดสอบ ----
result = graph.invoke({
    "messages": [{"role": "user", "content": "อากาศกรุงเทพวันนี้เป็นยังไง?"}]
})

for msg in result["messages"]:
    if msg.type == "human":
        print(f"👤 {msg.content}")
    elif msg.type == "ai" and msg.content:
        print(f"🤖 {msg.content}")
    elif msg.type == "ai" and hasattr(msg, 'tool_calls') and msg.tool_calls:
        print(f"🔧 เรียก: {msg.tool_calls[0]['name']}({msg.tool_calls[0]['args']})")
    elif msg.type == "tool":
        print(f"📦 ผล: {msg.content}")
```

**Output:**
```
👤 อากาศกรุงเทพวันนี้เป็นยังไง?
🔧 เรียก: get_weather({'city': 'กรุงเทพ'})
📦 ผล: 35°C แดดจัด
🤖 อากาศที่กรุงเทพวันนี้ 35°C แดดจัดครับ แนะนำดื่มน้ำเยอะๆ
```

**Flow:**
```
START → agent → (มี tool_calls?) → Yes → tools → agent → (มี tool_calls?) → No → END
```

---

## 4.5 Conditional Edges — แยกทาง

```python
# file: 24_branching.py

from typing import Literal

class RoutingState(TypedDict):
    messages: Annotated[list, add_messages]
    category: str

def classify(state: RoutingState):
    """จำแนกหมวดหมู่คำถาม"""
    question = state["messages"][-1].content
    response = model.invoke(
        f"จำแนกคำถามนี้เป็น: tech หรือ general\nคำถาม: {question}\nตอบแค่คำเดียว:"
    )
    cat = "tech" if "tech" in response.content.lower() else "general"
    return {"category": cat}

def tech_bot(state: RoutingState):
    """ตอบเรื่อง tech"""
    msgs = [{"role": "system", "content": "คุณคือ IT expert"}, *state["messages"]]
    return {"messages": [model.invoke(msgs)]}

def general_bot(state: RoutingState):
    """ตอบเรื่องทั่วไป"""
    msgs = [{"role": "system", "content": "คุณคือผู้ช่วยทั่วไป"}, *state["messages"]]
    return {"messages": [model.invoke(msgs)]}

# ---- Routing Function ----
def route(state: RoutingState) -> Literal["tech_bot", "general_bot"]:
    #                               ▲
    #                          Literal = บอกว่า return ได้แค่ค่าเหล่านี้
    if state["category"] == "tech":
        return "tech_bot"
    return "general_bot"

# ---- Build ----
builder = StateGraph(RoutingState)
builder.add_node("classify", classify)
builder.add_node("tech_bot", tech_bot)
builder.add_node("general_bot", general_bot)

builder.add_edge(START, "classify")
builder.add_conditional_edges("classify", route)
#                                         ▲
#                               function ที่ return ชื่อ Node ถัดไป

builder.add_edge("tech_bot", END)
builder.add_edge("general_bot", END)

graph = builder.compile()

# Flow:
# START → classify → (tech?) → tech_bot → END
#                   → (general?) → general_bot → END
```

---

## 4.6 Checkpointing — จำบทสนทนาข้าม invoke

```python
# file: 25_memory.py

from langgraph.checkpoint.memory import MemorySaver

# ---- Compile พร้อม Checkpointer ----
memory = MemorySaver()
#        ▲
#   MemorySaver = เก็บ state ใน RAM
#   (production ใช้ SqliteSaver หรือ PostgresSaver)

graph = builder.compile(checkpointer=memory)

# ---- ใช้งาน: ต้องระบุ thread_id ----
config = {"configurable": {"thread_id": "user-001"}}
#                           ▲
#                     thread_id = แยก session
#                     user ต่าง thread → ไม่เห็นข้อมูลกัน

# รอบ 1
r1 = graph.invoke(
    {"messages": [{"role": "user", "content": "ผมชื่อ สมชาย"}]},
    config   # ← ต้องส่ง config ทุกครั้ง
)
print(r1["messages"][-1].content)  # "สวัสดีครับ คุณสมชาย!"

# รอบ 2 (AI จำได้!)
r2 = graph.invoke(
    {"messages": [{"role": "user", "content": "ผมชื่ออะไร?"}]},
    config   # ← thread_id เดิม → จำบทสนทนาเก่าได้
)
print(r2["messages"][-1].content)  # "คุณชื่อ สมชาย ครับ"

# Thread ใหม่ (AI ไม่จำ)
config_new = {"configurable": {"thread_id": "user-002"}}
r3 = graph.invoke(
    {"messages": [{"role": "user", "content": "ผมชื่ออะไร?"}]},
    config_new  # ← thread_id ใหม่ → ไม่รู้จัก
)
print(r3["messages"][-1].content)  # "ผมไม่ทราบชื่อของคุณครับ"
```

---

## 4.7 Human-in-the-Loop — หยุดให้คนอนุมัติ

```python
# file: 26_human_approval.py

graph = builder.compile(
    checkpointer=memory,
    interrupt_before=["tools"],  # ← หยุดก่อนรัน tools!
    #  ▲
    # interrupt_before = หยุด graph ก่อนเข้า node ที่ระบุ
    # ให้คนมาตรวจดูก่อนว่า AI จะเรียก tool อะไร
)

config = {"configurable": {"thread_id": "approval-1"}}

# Step 1: AI ตัดสินใจจะเรียก tool แต่ถูกหยุด
result = graph.invoke(
    {"messages": [{"role": "user", "content": "ดูอากาศกรุงเทพ"}]},
    config
)

# ดูว่า AI ต้องการทำอะไร
tool_call = result["messages"][-1].tool_calls[0]
print(f"AI ต้องการเรียก: {tool_call['name']}({tool_call['args']})")

# Step 2: คนตัดสินใจ
approval = input("อนุมัติ? (y/n): ")

if approval == "y":
    # ทำต่อจากจุดที่หยุดไว้
    result = graph.invoke(None, config)  # None = ไม่เพิ่ม input ใหม่ แค่ทำต่อ
    print(f"คำตอบ: {result['messages'][-1].content}")
else:
    print("ยกเลิก")
```

---

## สรุป Step 4

```
✅ เข้าใจ State, Node, Edge, Graph
✅ สร้าง Graph แบบเส้นตรง (A → B → C)
✅ เพิ่ม Tools ใน Graph (Agent loop)
✅ สร้าง Conditional Edges (แยกทาง)
✅ Checkpointing (จำบทสนทนาข้าม invoke)
✅ Human-in-the-Loop (หยุดให้คนอนุมัติ)
```

### เส้นทางต่อไป
```
Step 5 → Advanced RAG (Hybrid Search, Re-ranking)
Step 6 → Multi-Agent (หลาย Agent ทำงานร่วมกัน)
Step 7 → Evaluation (วัดคุณภาพ)
Step 8 → Deployment (Deploy ขึ้น Cloud)
```
