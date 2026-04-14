---
title: "Tool Calls / Function Calling — พื้นฐานฉบับสมบูรณ์"
type: source
source_file: raw/notes/ai/agents/tool-calls.md
tags: [tool-calls, function-calling, api, llm, agents]
related: []
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../../raw/notes/ai/agents/tool-calls.md|Original file]]

## สรุป

คู่มือครบถ้วนเกี่ยวกับ Tool Calls / Function Calling — กลไกที่ทำให้ LLM สามารถเรียกใช้ functions ภายนอกได้ ครอบคลุมตั้งแต่ API level จนถึง implementation จริง รวมถึงความต่างระหว่าง Anthropic และ OpenAI APIs

## ประเด็นสำคัญ

### 1. ปัญหาที่ Tool Calls แก้

**LLM ปกติ (ไม่มี tools):**
- รู้เฉพาะสิ่งที่ train มาจนถึง cutoff date
- ข้อมูล stale หรือ hallucinate ได้
- ไม่สามารถเข้าถึงข้อมูล real-time, database, API ภายนอก

**LLM ที่มี Tool Calls:**
- Generate tool call → โปรแกรมเรียก API จริง → ส่งผลกลับให้ LLM → LLM ตอบด้วยข้อมูลจริง

> [!important] หลักการสำคัญ
> **LLM ไม่ได้ execute function เอง** — มันแค่บอกว่า "call อะไร ด้วย argument อะไร"
> **โปรแกรมของเรา** เป็นคนรัน function จริงและส่งผลกลับ

### 2. มันทำงานยังไง — Flow ทั้งหมด

```
1. User asks question
2. Send to LLM API (with tool definitions)
3. LLM generates JSON output → tool_use response
4. Parse tool call from response
5. Execute the actual function
6. Send result back to LLM
7. LLM reads result → generates final answer
8. Return answer to user
```

**แต่ละ tool call = 1 round trip กับ LLM API**
- ถ้า LLM call tool 3 ครั้ง = **4 LLM calls** รวม (3 tool + 1 final)

### 3. โครงสร้าง Tool Definition

```python
{
    "name": "search_documents",          # ← ชื่อ function (LLM เลือกโดยอ่านชื่อ)
    "description": """
        ค้นหาเอกสารจาก knowledge base ด้วย semantic search
        ใช้เมื่อต้องการข้อมูลจากเอกสารภายใน
        ไม่เหมาะกับข้อมูล real-time หรือข้อมูลส่วนตัวของ user
    """,
    # ↑ สำคัญมาก — LLM อ่าน description เพื่อตัดสินว่าจะ call tool นี้มั้ย

    "input_schema": {                    # ← JSON Schema มาตรฐาน
        "type": "object",
        "properties": {
            "query": {"type": "string", "description": "คำค้นหา"},
            "top_k": {"type": "integer", "description": "จำนวน docs", "default": 3}
        },
        "required": ["query"]
    }
}
```

> [!warning] Tool Description คือทุกอย่าง
> LLM เลือก tool โดยอ่าน description → **description ต้องชัดมาก**
> - แย่: "search documents"
> - ดี: "ค้นหาเอกสารจาก HR knowledge base ใช้เมื่อ: ถามเรื่องนโยบาย วันลา สวัสดิการ"

### 4. Parallel Tool Calls

LLM สามารถ call หลาย tools ใน response เดียว ถ้า **independent กัน**

```python
# Response จาก LLM อาจมีหลาย ToolUseBlock พร้อมกัน
response.content = [
    ToolUseBlock(id="t1", name="get_gold_price", input={"date": "today"}),
    ToolUseBlock(id="t2", name="get_usd_thb_rate", input={"date": "today"}),
    ToolUseBlock(id="t3", name="get_silver_price", input={"date": "today"})
]

# Execute ทั้งหมด parallel
results = await asyncio.gather(
    get_gold_price(date="today"),
    get_usd_thb_rate(date="today"),
    get_silver_price(date="today")
)
```

### 5. Error Handling ใน Tool Results

```python
{
    "type": "tool_result",
    "tool_use_id": "t1",
    "content": "Error: API timeout after 30s",
    "is_error": True    # ← บอก LLM ว่านี่คือ error
}
```

LLM จะ handle gracefully — เช่น "ไม่สามารถดึงราคาทองได้ในขณะนี้ กรุณาลองใหม่อีกครั้ง"

### 6. Tool Choice Control

```python
# บังคับให้ LLM ใช้ tool เสมอ
tool_choice={"type": "any"}

# บังคับให้ใช้ tool เฉพาะนี้
tool_choice={"type": "tool", "name": "search_documents"}

# ปล่อยให้ LLM ตัดสินใจเอง (default)
tool_choice={"type": "auto"}

# ห้ามใช้ tool เลย
tool_choice={"type": "none"}
```

### 7. เปรียบเทียบ APIs — Anthropic vs OpenAI

|  aspect  | Anthropic | OpenAI |
|---|---|---|
| Tool def key | `input_schema` | `parameters` |
| Tool result role | `user` | `tool` |
| Arguments format | dict (parsed) | JSON string (ต้อง `json.loads`) |
| Response check | `block.type == "tool_use"` | `choice.finish_reason == "tool_calls"` |

### 8. Framework ต่างๆ ห่อหุ้มยังไง

**สิ่งที่ Framework ทำให้:**

| งาน | Raw API | Framework |
|---|---|---|
| สร้าง tool schema | เขียนเอง | auto จาก docstring/type hints |
| Parse tool call | เขียนเอง | อัตโนมัติ |
| Execute function | เขียนเอง | อัตโนมัติ |
| ส่ง result กลับ | เขียนเอง | อัตโนมัติ |
| Loop จนจบ | เขียนเอง | อัตโนมัติ |

**LangChain:**
```python
from langchain_core.tools import tool

@tool
def search_hr_docs(query: str, top_k: int = 3) -> str:
    """ค้นหาเอกสารจาก HR knowledge base"""  # ← docstring = description
    return str(do_vector_search(query, top_k))
```

### 9. Tool Call กับ Agent — ความสัมพันธ์

```
Tool Call ≠ Agent

Tool Call คือ:
  - mechanism ที่ LLM request ให้ execute function
  - 1 request → 1 response (อาจมีหลาย tools)
  - stateless (ไม่มี loop built-in)

Agent คือ:
  - โปรแกรมที่ใช้ tool calls ซ้ำๆ ในลักษณะ loop
  - มี planning/reasoning ระหว่างขั้นตอน
  - stateful (จำ conversation history)

Agent = Tool Calls + Loop + Memory + Reasoning
```

### 10. ข้อควรระวัง

**อย่า Trust Input จาก LLM โดยตรง**

```python
# อันตราย — ถ้า LLM hallucinate filename
def read_file(filename: str) -> str:
    return open(filename).read()   # ← path traversal!

# ปลอดภัย — validate ก่อนเสมอ
def read_file(filename: str) -> str:
    allowed_dir = Path("/data/docs")
    target = (allowed_dir / filename).resolve()
    if not str(target).startswith(str(allowed_dir)):
        raise ValueError("Path traversal detected")
    return target.read_text()
```

**จำกัด Max Iterations**

```python
MAX_ITERATIONS = 10
for i in range(MAX_ITERATIONS):
    response = call_llm(messages)
    if response.stop_reason == "end_turn":
        break
    # handle tool calls...
else:
    raise Exception("Agent loop exceeded max iterations")
```

## ข้อมูล / หลักฐาน ที่น่าสนใจ

### API Response Structure จริง (Anthropic)

**Step 2 — LLM ตอบกลับด้วย tool call:**
```python
response.stop_reason    # "tool_use"  ← ไม่ใช่ "end_turn"
response.content
# [
#   TextBlock(text="ผมจะดึงราคาทองให้"),
#   ToolUseBlock(
#     id="toolu_01XYZ",
#     name="get_gold_price",
#     input={"date": "today", "unit": "baht_per_baht"}
#   )
# ]
```

**Step 4 — ส่งผลกลับให้ LLM:**
```python
{
    "role": "user",  # ← Anthropic: role เป็น "user" เสมอ
    "content": [
        {
            "type": "tool_result",
            "tool_use_id": tool_use.id,  # ← ต้อง match กับ id
            "content": '{"price": 42500, "date": "2026-04-10"}'
        }
    ]
}
```

### Mental Model

```
LLM  = ฉลาด แต่ไม่รู้อะไรนอกจากที่ train มา
      รัน code ไม่ได้
      เข้าถึง internet/database ไม่ได้

Tool Call  = "ประตู" ที่ LLM ขอเปิดเพื่อเข้าถึงโลกภายนอก

เรา (App)  = คนที่เปิดประตูให้ (execute function จริง)
           คนที่ส่งผลกลับให้ LLM
           คนที่ควบคุม loop ทั้งหมด

LLM ไม่ได้ "ทำ" tool call — มันแค่ "ขอให้ทำ"
```

## Concepts ที่เกี่ยวข้อง

- [[Function Calling]] - Tool use mechanism
- [[Agent]] - LLM + Tools + Loop + Memory
- [[ReAct]] - Pattern ที่ใช้ tool calls
- [[Tool Definition]] - วิธีกำหนด tools ให้ LLM
- [[API Design]] - Anthropic vs OpenAI differences
- [[Error Handling]] - จัดการ errors ใน tool calls
