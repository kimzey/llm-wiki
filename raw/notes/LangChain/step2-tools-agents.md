# Step 2: Tools & Agents
## สร้างเครื่องมือให้ AI ใช้ + สร้าง Agent ที่คิดเองได้

> Step นี้จะทำให้ AI ไม่ใช่แค่ "ตอบคำถาม" แต่สามารถ "ลงมือทำ" ได้ เช่น ค้นข้อมูล คำนวณ เรียก API

---

## 2.1 ทำไม AI ถึงต้องมี Tools?

```
ไม่มี Tools:
  User: "อากาศวันนี้เป็นยังไง?"
  AI:   "ขออภัย ผมไม่สามารถดูสภาพอากาศปัจจุบันได้" ← ทำไม่ได้!

มี Tools:
  User: "อากาศวันนี้เป็นยังไง?"
  AI:   (คิด) → ต้องใช้ tool "get_weather"
        (ทำ) → เรียก get_weather("กรุงเทพ")
        (ได้) → "35°C แดดจัด"
        (ตอบ) → "วันนี้กรุงเทพอุณหภูมิ 35°C แดดจัดครับ" ← ทำได้!
```

---

## 2.2 สร้าง Tool ตัวแรก — @tool Decorator

```python
# file: 13_first_tool.py

from dotenv import load_dotenv
from langchain.tools import tool

load_dotenv()

# ---- สร้าง Tool ด้วย @tool ----
@tool
def add(a: float, b: float) -> float:
    """บวกเลข 2 จำนวนเข้าด้วยกัน"""
    return a + b

# อธิบายทุกส่วน:
#
# @tool                    ← decorator ที่แปลง function เป็น Tool
#
# def add(                 ← ชื่อ function = ชื่อ tool
#     a: float,            ← parameter + type hint (จำเป็น!)
#     b: float,            ← AI จะรู้ว่าต้องใส่อะไร จากตรงนี้
# ) -> float:              ← return type
#
# """บวกเลข 2 จำนวน..."""  ← docstring (จำเป็นมาก!)
#                             AI จะอ่านตรงนี้เพื่อตัดสินใจว่า
#                             จะใช้ tool นี้ในสถานการณ์ไหน
#
# return a + b             ← logic จริงๆ ของ tool

# ---- ดูข้อมูลของ Tool ----
print(f"ชื่อ Tool:  {add.name}")          # "add"
print(f"คำอธิบาย:  {add.description}")    # "บวกเลข 2 จำนวนเข้าด้วยกัน"
print(f"Schema:    {add.args_schema.model_json_schema()}")
# {'properties': {'a': {'type': 'number'}, 'b': {'type': 'number'}}, ...}

# ---- ทดสอบ Tool ตรงๆ (ไม่ผ่าน AI) ----
result = add.invoke({"a": 10, "b": 20})
print(f"10 + 20 = {result}")  # 30
```

### สิ่งสำคัญที่ต้องมีใน Tool

```python
# ❌ ไม่ดี — ไม่มี type hint, ไม่มี docstring
@tool
def search(q):
    return "ผลลัพธ์"

# ✅ ดี — มีครบ
@tool
def search(query: str) -> str:
    """ค้นหาข้อมูลจากฐานข้อมูลของบริษัท ใช้เมื่อต้องการหาข้อมูลพนักงาน นโยบาย หรือสวัสดิการ"""
    return "ผลลัพธ์"

# ทำไม docstring สำคัญ?
# → เพราะ AI จะอ่าน docstring เพื่อตัดสินใจว่าจะใช้ tool นี้เมื่อไหร่
# → ถ้าเขียนไม่ดี AI จะไม่รู้ว่าควรเรียก tool นี้ตอนไหน
```

---

## 2.3 สร้าง Tools หลายตัว

```python
# file: 14_multiple_tools.py

from langchain.tools import tool

# ---- Tool 1: คำนวณ ----
@tool
def calculate(expression: str) -> str:
    """คำนวณนิพจน์ทางคณิตศาสตร์ เช่น '100 * 25' หรือ '(50 + 30) * 2'
    ใช้เมื่อต้องการคำนวณตัวเลข"""
    try:
        result = eval(expression)
        #        ▲
        #   eval() รัน string เป็น Python expression
        #   "100 * 25" → 2500
        return f"{expression} = {result}"
    except Exception as e:
        return f"คำนวณไม่ได้: {e}"

# ---- Tool 2: ค้นหาข้อมูลบริษัท ----
@tool
def search_company_info(query: str) -> str:
    """ค้นหาข้อมูลภายในบริษัท เช่น นโยบาย สวัสดิการ ข้อมูลพนักงาน
    ใช้เมื่อมีคำถามเกี่ยวกับบริษัท"""

    # Mock database (ในงานจริงจะเชื่อม DB หรือ Vector Store)
    database = {
        "ลาพักร้อน": "พนักงานมีสิทธิ์ลาพักร้อน 15 วัน/ปี",
        "ค่าอาหาร": "ค่าอาหาร 100 บาท/วันทำการ",
        "ค่าเดินทาง": "ค่าเดินทาง 2,500 บาท/เดือน",
        "เวลาทำงาน": "จันทร์-ศุกร์ 9:00-18:00",
        "โบนัส": "โบนัสประจำปี 0-4 เดือน ตามผลงาน",
    }

    for key, value in database.items():
        if key in query:
            return f"📋 ข้อมูลที่พบ: {value}"
    return "❌ ไม่พบข้อมูลที่ค้นหา"

# ---- Tool 3: ดูเวลา ----
@tool
def get_current_time() -> str:
    """ดูวันที่และเวลาปัจจุบัน"""
    from datetime import datetime
    now = datetime.now()
    return now.strftime("วันที่ %d/%m/%Y เวลา %H:%M:%S")

# ทดสอบ
print(calculate.invoke({"expression": "100 * 22"}))
print(search_company_info.invoke({"query": "ลาพักร้อน"}))
print(get_current_time.invoke({}))
```

**Output:**
```
100 * 22 = 2200
📋 ข้อมูลที่พบ: พนักงานมีสิทธิ์ลาพักร้อน 15 วัน/ปี
วันที่ 10/04/2026 เวลา 14:30:25
```

---

## 2.4 bind_tools — ให้ AI รู้จัก Tools

```python
# file: 15_bind_tools.py

from dotenv import load_dotenv
from langchain_google_genai import ChatGoogleGenerativeAI

load_dotenv()
model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

# ---- ให้ model รู้จัก tools ----
tools = [calculate, search_company_info, get_current_time]

model_with_tools = model.bind_tools(tools)
#                        ▲
#                   bind_tools() = "สอน" ให้ model รู้ว่ามี tools อะไรบ้าง
#                   model จะอ่าน name + description + parameters ของแต่ละ tool
#                   แล้วตัดสินใจเองว่าจะเรียก tool ไหน

# ---- ทดสอบ: ถามคำถามที่ต้องใช้ tool ----
response = model_with_tools.invoke("ค่าอาหาร 22 วัน เท่ากับเท่าไหร่?")

# ดูว่า AI ต้องการเรียก tool อะไร
print(f"Content: '{response.content}'")    # อาจว่างเปล่า!
print(f"Tool calls: {response.tool_calls}") # มี tool call!
```

**Output:**
```
Content: ''        ← ว่างเปล่า! เพราะ AI ยังไม่ตอบ แค่บอกว่าอยากเรียก tool
Tool calls: [
    {
        'name': 'search_company_info',
        'args': {'query': 'ค่าอาหาร'},
        'id': 'call_abc123'
    }
]
```

### อธิบาย Tool Calling Flow

```
User: "ค่าอาหาร 22 วัน เท่ากับเท่าไหร่?"
                    │
                    ▼
         AI คิด (Reasoning):
         "ต้องรู้ก่อนว่าค่าอาหารวันละเท่าไหร่"
         "ต้องใช้ search_company_info"
                    │
                    ▼
         AI ตอบ: tool_calls = [
           {name: "search_company_info", args: {query: "ค่าอาหาร"}}
         ]
         content = ""  ← ยังไม่ตอบ! รอผลจาก tool ก่อน
                    │
                    ▼
         เรา: ต้องรัน tool แล้วส่งผลกลับ (ดูหัวข้อถัดไป)
```

---

## 2.5 รัน Tool ด้วยตัวเอง (Manual Execution)

```python
# file: 16_manual_tool.py

from langchain.messages import HumanMessage, ToolMessage

# Step 1: ส่งคำถาม
response = model_with_tools.invoke([
    HumanMessage(content="ค่าอาหารวันละเท่าไหร่?")
])

print("Step 1 - AI ต้องการเรียก tool:")
print(f"  Tool: {response.tool_calls[0]['name']}")
print(f"  Args: {response.tool_calls[0]['args']}")

# Step 2: รัน tool ด้วยตัวเอง
tool_call = response.tool_calls[0]
#           ▲
#      ดึง tool call แรกออกมา

# หา tool ที่ตรงชื่อ
tool_map = {t.name: t for t in tools}
#           ▲
#   สร้าง dict: {"calculate": calculate_tool, "search_company_info": search_tool, ...}

tool_to_use = tool_map[tool_call["name"]]
tool_result = tool_to_use.invoke(tool_call["args"])
#                                ▲
#                         ส่ง args ที่ AI กำหนดมา

print(f"\nStep 2 - ผลจาก tool: {tool_result}")

# Step 3: ส่งผลกลับให้ AI สรุปคำตอบ
final_response = model_with_tools.invoke([
    HumanMessage(content="ค่าอาหารวันละเท่าไหร่?"),
    response,                              # ← AIMessage ที่มี tool_calls
    ToolMessage(
        content=str(tool_result),          # ← ผลลัพธ์จาก tool
        tool_call_id=tool_call["id"],      # ← ID ที่ต้องตรงกัน
    ),
])

print(f"\nStep 3 - คำตอบสุดท้าย: {final_response.content}")
```

**Output:**
```
Step 1 - AI ต้องการเรียก tool:
  Tool: search_company_info
  Args: {'query': 'ค่าอาหาร'}

Step 2 - ผลจาก tool: 📋 ข้อมูลที่พบ: ค่าอาหาร 100 บาท/วันทำการ

Step 3 - คำตอบสุดท้าย: ค่าอาหารวันละ 100 บาทครับ
```

---

## 2.6 Agent — ให้ AI ทำทุกอย่างเองอัตโนมัติ

### ปัญหาของ Manual Execution

```
❌ ต้องเขียน code รัน tool เอง
❌ ถ้า AI ต้องเรียก tool หลายตัว → ต้องเขียน loop เอง
❌ ถ้า AI ต้องเรียก tool → ดูผล → เรียก tool อีกตัว → ยุ่งมาก

✅ Agent จัดการให้หมด:
   - AI ตัดสินใจเรียก tool เอง
   - รัน tool อัตโนมัติ
   - ดูผล → ตัดสินใจต่อ → จนได้คำตอบสุดท้าย
```

### สร้าง Agent ด้วย create_agent

```python
# file: 17_agent.py

from dotenv import load_dotenv
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain.agents import create_agent
from langchain.tools import tool

load_dotenv()

# ---- Tools (เหมือนเดิม) ----
@tool
def calculate(expression: str) -> str:
    """คำนวณนิพจน์ทางคณิตศาสตร์"""
    try:
        return f"{expression} = {eval(expression)}"
    except:
        return "คำนวณไม่ได้"

@tool
def search_company_info(query: str) -> str:
    """ค้นหาข้อมูลบริษัท เช่น นโยบาย สวัสดิการ"""
    db = {
        "ค่าอาหาร": "ค่าอาหาร 100 บาท/วันทำการ",
        "ค่าเดินทาง": "ค่าเดินทาง 2,500 บาท/เดือน",
        "ลาพักร้อน": "ลาพักร้อน 15 วัน/ปี",
    }
    for key, value in db.items():
        if key in query:
            return value
    return "ไม่พบข้อมูล"

# ---- สร้าง Model ----
model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)
#                                                              ▲
#                                              temperature=0 สำหรับ Agent!
#                                              ลดความ "สุ่ม" ให้ตัดสินใจแม่นยำ

# ---- สร้าง Agent ----
agent = create_agent(
    model=model,                    # LLM ที่จะใช้
    tools=[calculate, search_company_info],  # tools ที่ Agent ใช้ได้
    system_prompt="""คุณคือผู้ช่วย HR ของบริษัท ABC
กฎ:
1. ค้นหาข้อมูลจากฐานข้อมูลก่อนตอบเสมอ
2. ถ้าต้องคำนวณ ให้ใช้เครื่องมือคำนวณ
3. ตอบเป็นภาษาไทย สุภาพ""",
)

# create_agent คืออะไร?
# = สร้าง "Agent" ที่จะ:
#   1. รับคำถามจาก user
#   2. คิดว่าต้องใช้ tool ไหน (Reasoning)
#   3. เรียก tool อัตโนมัติ (Acting)
#   4. ดูผลลัพธ์ → คิดต่อ → เรียก tool อื่น (ถ้าจำเป็น)
#   5. สรุปคำตอบสุดท้าย

# ---- ใช้งาน Agent ----
result = agent.invoke({
    "messages": [{"role": "user", "content": "ค่าอาหาร 22 วัน รวมค่าเดินทาง เดือนนึงเท่าไหร่?"}]
})
#       ▲
#  agent.invoke() รับ dict ที่มี key "messages"
#  messages เป็น list ของข้อความ

# ---- ดูผลลัพธ์ ----
# result["messages"] = ทุก message ที่เกิดขึ้น (รวม tool calls)
for msg in result["messages"]:
    role = msg.type  # human, ai, tool
    content = msg.content if isinstance(msg.content, str) else str(msg.content)

    if role == "human":
        print(f"👤 User: {content}")
    elif role == "ai" and content:
        print(f"🤖 AI: {content}")
    elif role == "ai" and not content and hasattr(msg, 'tool_calls') and msg.tool_calls:
        for tc in msg.tool_calls:
            print(f"🔧 AI เรียก tool: {tc['name']}({tc['args']})")
    elif role == "tool":
        print(f"📦 Tool result: {content}")
```

**Output:**
```
👤 User: ค่าอาหาร 22 วัน รวมค่าเดินทาง เดือนนึงเท่าไหร่?
🔧 AI เรียก tool: search_company_info({'query': 'ค่าอาหาร'})
📦 Tool result: ค่าอาหาร 100 บาท/วันทำการ
🔧 AI เรียก tool: search_company_info({'query': 'ค่าเดินทาง'})
📦 Tool result: ค่าเดินทาง 2,500 บาท/เดือน
🔧 AI เรียก tool: calculate({'expression': '100 * 22 + 2500'})
📦 Tool result: 100 * 22 + 2500 = 4700
🤖 AI: ค่าอาหาร 22 วัน = 2,200 บาท + ค่าเดินทาง 2,500 บาท
       รวมเดือนละ 4,700 บาทครับ
```

### อธิบาย Flow ที่เกิดขึ้น (ReAct Loop)

```
รอบ 1:
  Reasoning: "ต้องรู้ค่าอาหารก่อน"
  Acting:    เรียก search_company_info("ค่าอาหาร")
  Result:    "ค่าอาหาร 100 บาท/วัน"

รอบ 2:
  Reasoning: "ต้องรู้ค่าเดินทางด้วย"
  Acting:    เรียก search_company_info("ค่าเดินทาง")
  Result:    "ค่าเดินทาง 2,500 บาท/เดือน"

รอบ 3:
  Reasoning: "ต้องคำนวณ 100*22 + 2500"
  Acting:    เรียก calculate("100 * 22 + 2500")
  Result:    "4700"

รอบ 4:
  Reasoning: "ได้ข้อมูลครบแล้ว สรุปคำตอบ"
  Acting:    ไม่เรียก tool → ตอบ user
  Result:    "รวมเดือนละ 4,700 บาทครับ"
```

---

## 2.7 ทำ Chatbot ที่จำบทสนทนาได้

```python
# file: 18_chatbot.py

from dotenv import load_dotenv
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain.agents import create_agent

load_dotenv()

model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

agent = create_agent(
    model=model,
    tools=[calculate, search_company_info],
    system_prompt="คุณคือผู้ช่วย HR ของบริษัท ABC ตอบสั้นกระชับ",
)

# เก็บประวัติบทสนทนา
history = []

def chat(user_input: str) -> str:
    """ส่งข้อความแล้วได้คำตอบ พร้อมจำประวัติ"""
    # เพิ่ม message ใหม่เข้า history
    history.append({"role": "user", "content": user_input})

    # ส่งทั้ง history ให้ agent
    result = agent.invoke({"messages": history})

    # ดึงคำตอบสุดท้าย
    ai_answer = result["messages"][-1].content

    # เก็บคำตอบเข้า history
    history.append({"role": "assistant", "content": ai_answer})

    return ai_answer

# ทดสอบ
print(chat("ค่าอาหารวันละเท่าไหร่?"))
print(chat("แล้วถ้าทำงาน 22 วัน เท่าไหร่?"))  # AI จำได้ว่ากำลังคุยเรื่องค่าอาหาร
print(chat("รวมค่าเดินทางด้วยเท่าไหร่?"))      # AI จำได้อีก
```

**Output:**
```
ค่าอาหารวันละ 100 บาทครับ
ค่าอาหาร 22 วัน = 2,200 บาทครับ
ค่าอาหาร 2,200 + ค่าเดินทาง 2,500 = รวม 4,700 บาท/เดือนครับ
```

---

## สรุป Step 2 — สิ่งที่ทำได้แล้ว

```
✅ สร้าง Tool ด้วย @tool decorator
✅ เข้าใจ type hints + docstring สำคัญอย่างไร
✅ bind_tools() ให้ model รู้จัก tools
✅ เข้าใจ Tool Calling Flow (AI ตัดสินใจ → เรียก tool → ได้ผล)
✅ รัน tool ด้วยตัวเอง (Manual execution)
✅ สร้าง Agent ด้วย create_agent (อัตโนมัติทั้งหมด)
✅ เข้าใจ ReAct Loop (Reasoning + Acting)
✅ Agent เรียก tools หลายตัวต่อเนื่องได้
✅ ทำ Chatbot ที่จำบทสนทนาได้
```

### ไป Step 3 ต่อ → RAG (ให้ AI ตอบจากเอกสารของเรา)
