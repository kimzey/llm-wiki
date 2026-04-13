---
title: "Step 4: LangGraph — ควบคุม Flow ด้วย Graph"
type: source
source_file: raw/notes/LangChain/step4-langgraph-tutorial.md
tags: [langchain, langgraph, tutorial, workflow, state]
related: []
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/LangChain/step4-langgraph-tutorial.md|Original file]]

## สรุป

บทช่วยสอน LangGraph สอนวิธีสร้าง Agent ที่มี workflow ซับซ้อน ตั้งแต่พื้นฐาน เนื้อหาครอบคลุม State, Nodes, Edges, Graph, การเพิ่ม Tools, Conditional Edges, Checkpointing, และ Human-in-the-Loop

## ประเด็นสำคัญ

### LangChain Agent vs LangGraph
- **LangChain Agent (`create_agent`)**: เหมือน "พนักงาน 1 คน" ที่ทำทุกอย่างเอง → ดีสำหรับงานง่ายๆ แต่ควบคุมยาก
- **LangGraph**: เหมือน "แผนผังการทำงาน" ที่กำหนดว่าใครทำอะไร → ส่งต่อให้ใคร → ตรวจสอบอะไร → จบตอนไหน → ควบคุมได้ทุกจุด

### องค์ประกอบ 4 อย่าง
1. **State** = "กระดาษจดบันทึก" ที่ทุกขั้นตอนอ่าน/เขียนได้
2. **Node** = "ขั้นตอน" (เช่น ถาม AI, ค้นหาข้อมูล, ตรวจสอบ)
3. **Edge** = "ลูกศร" เชื่อมขั้นตอน (A → B)
4. **Graph** = "แผนผัง" ที่รวมทุกอย่างเข้าด้วยกัน

### State และ Reducer
- `Annotated[list, add_messages]`:
  - `list` = เก็บเป็น list
  - `add_messages` = "reducer" (วิธีรวมข้อมูล)
  - reducer คือ กฎว่าเมื่อ Node return ข้อมูลใหม่ จะรวมกับข้อมูลเก่าอย่างไร
  - `add_messages` = APPEND (เพิ่มต่อท้าย)
  - ถ้าไม่มี reducer = REPLACE (ทับเลย)

### Node (function)
- รับ State เป็น input
- Return dict → ค่าจะถูกอัปเดตเข้า State
- `return {"messages": [response]}` → `[response]` จะถูก APPEND เข้า list เดิม

### Graph Building
```python
builder = StateGraph(State)
builder.add_node("chatbot", chatbot)  # ชื่อ Node, function
builder.add_edge(START, "chatbot")    # START → chatbot
builder.add_edge("chatbot", END)      # chatbot → END
graph = builder.compile()
```

### ToolNode
- Node สำเร็จรูปที่:
  1. อ่าน `tool_calls` จาก AIMessage ล่าสุด
  2. รัน tools ที่ถูกเรียก
  3. Return ผลเป็น ToolMessage

### Conditional Edges
- `add_conditional_edges`: เส้นเชื่อมที่ไปคนละทาง ขึ้นอยู่กับเงื่อนไข
- `tools_condition`: function สำเร็จรูป
  - ถ้า AI มี `tool_calls` → ไปที่ "tools"
  - ถ้าไม่มี → ไปที่ END
- Custom routing function: return ชื่อ Node ถัดไป

### Routing Function
```python
def route(state: RoutingState) -> Literal["tech_bot", "general_bot"]:
    if state["category"] == "tech":
        return "tech_bot"
    return "general_bot"
```
- `Literal` = บอกว่า return ได้แค่ค่าเหล่านี้

### Checkpointing (จำบทสนทนาข้าม invoke)
```python
memory = MemorySaver()  # เก็บ state ใน RAM
graph = builder.compile(checkpointer=memory)

config = {"configurable": {"thread_id": "user-001"}}
# thread_id = แยก session
```
- user ต่าง thread → ไม่เห็นข้อมูลกัน

### Human-in-the-Loop
```python
graph = builder.compile(
    checkpointer=memory,
    interrupt_before=["tools"],  # หยุดก่อนรัน tools!
)
```
- `interrupt_before`: หยุด graph ก่อนเข้า node ที่ระบุ
- ให้คนมาตรวจดูก่อนว่า AI จะเรียก tool อะไร

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- Flow ของ Agent loop:
  ```
  START → agent → (มี tool_calls?) → Yes → tools → agent → (มี tool_calls?) → No → END
  ```
- Thread ใหม่ (AI ไม่จำ): thread_id ต่างกัน → AI ไม่รู้จัก
- `invoke(None, config)`: ทำต่อจากจุดที่หยุดไว้ (หลังจาก human-in-the-loop)

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อมูลที่ขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/ai-agent.md|AI Agent]]
- [[wiki/concepts/agentic-rag.md|Agentic RAG]]
