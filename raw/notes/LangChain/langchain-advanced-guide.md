# LangChain Advanced Guide
## LangGraph, LangSmith & RAG อย่างละเอียด

> ต่อเนื่องจาก LangChain 101 — หัวข้อนี้จะพาไปรู้จักกับ LangGraph, LangSmith และ RAG (Retrieval-Augmented Generation)

---

## สารบัญ

1. [LangGraph — ควบคุม Flow ของ Agent อย่างละเอียด](#1-langgraph)
2. [LangSmith — ตรวจสอบและประเมินผล Agent](#2-langsmith)
3. [RAG — Retrieval-Augmented Generation](#3-rag)
4. [รวมทุกอย่างเข้าด้วยกัน — RAG Agent with LangGraph](#4-putting-it-all-together)

---

## 1. LangGraph

### LangGraph คืออะไร ?

LangGraph คือ library ที่สร้างขึ้นมาบน LangChain เพื่อให้เราสามารถ **ควบคุม Flow การทำงานของ Agent ในระดับ Low-level** ได้อย่างละเอียด โดยใช้แนวคิดของ **Graph** (กราฟ) ที่ประกอบด้วย:

- **Nodes (โหนด):** แต่ละ Node คือ "ขั้นตอนการทำงาน" หนึ่งขั้นตอน เช่น เรียก LLM, เรียก Tool, ตรวจสอบเงื่อนไข
- **Edges (เส้นเชื่อม):** กำหนดว่าจาก Node หนึ่งจะไปยัง Node ใดต่อ
- **Conditional Edges:** เส้นเชื่อมแบบมีเงื่อนไข เช่น ถ้า LLM ตัดสินใจเรียก Tool → ไปที่ Tool Node, ถ้าไม่ → ไปที่ End
- **State (สถานะ):** ข้อมูลที่ถูกส่งต่อระหว่าง Node ทำให้แต่ละขั้นตอนรู้บริบทของงาน

### ทำไมต้องใช้ LangGraph ?

| เคส | ใช้ LangChain (create_agent) | ใช้ LangGraph |
|------|------|------|
| Chatbot ง่ายๆ ถาม-ตอบ | ✅ เหมาะ | ❌ เกินความจำเป็น |
| Agent + Tools แบบพื้นฐาน | ✅ เหมาะ | ❌ เกินความจำเป็น |
| ต้องการ Branching Logic ซับซ้อน | ❌ ทำได้ยาก | ✅ เหมาะมาก |
| ต้องการ Human-in-the-loop | ❌ ไม่รองรับ | ✅ รองรับ |
| Multi-Agent ทำงานร่วมกัน | ❌ ทำได้ยาก | ✅ ออกแบบมาเพื่อสิ่งนี้ |
| ต้องการ Retry / Error handling ละเอียด | ⚠️ มี middleware แต่จำกัด | ✅ ควบคุมได้ทุกจุด |

### ติดตั้ง LangGraph

```bash
pip install -U langgraph langchain[google-genai]
```

### ตัวอย่างที่ 1: สร้าง Graph แบบง่าย — Chatbot ที่มี Tool

เริ่มจากตัวอย่างง่ายๆ ก่อน สร้าง Chatbot ที่มี Tool ค้นหาสภาพอากาศ:

```python
import os
from dotenv import load_dotenv
from typing import Annotated
from typing_extensions import TypedDict

from langchain_google_genai import ChatGoogleGenerativeAI
from langchain.tools import tool
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

load_dotenv()

# ---- 1. กำหนด State ----
# State คือข้อมูลที่จะถูกส่งต่อระหว่าง Node
# ในที่นี้เก็บแค่ messages โดยใช้ add_messages เป็น reducer
# (reducer = ฟังก์ชันที่กำหนดว่าจะรวมข้อมูลใหม่เข้ากับข้อมูลเก่าอย่างไร)
class State(TypedDict):
    messages: Annotated[list, add_messages]

# ---- 2. สร้าง Tools ----
@tool
def get_weather(city: str) -> str:
    """ค้นหาสภาพอากาศปัจจุบันของเมืองที่ระบุ"""
    # Mock data สำหรับตัวอย่าง
    weather_data = {
        "bangkok": "กรุงเทพฯ: อุณหภูมิ 35°C, แดดจัด, ความชื้น 70%",
        "chiangmai": "เชียงใหม่: อุณหภูมิ 28°C, มีเมฆบางส่วน, ความชื้น 55%",
        "phuket": "ภูเก็ต: อุณหภูมิ 32°C, ฝนตกเป็นระยะ, ความชื้น 80%",
    }
    city_key = city.lower().replace(" ", "")
    return weather_data.get(city_key, f"ไม่พบข้อมูลสภาพอากาศของ {city}")

tools = [get_weather]

# ---- 3. สร้างโมเดล และ bind tools เข้าไป ----
model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)
model_with_tools = model.bind_tools(tools)

# ---- 4. สร้าง Node Functions ----
# Node "chatbot": ให้โมเดลคิดและตัดสินใจ
def chatbot(state: State):
    response = model_with_tools.invoke(state["messages"])
    return {"messages": [response]}

# ---- 5. สร้าง Graph ----
graph_builder = StateGraph(State)

# เพิ่ม Nodes
graph_builder.add_node("chatbot", chatbot)
graph_builder.add_node("tools", ToolNode(tools=tools))

# เพิ่ม Edges
# จาก START → chatbot (เริ่มต้นที่ chatbot เสมอ)
graph_builder.add_edge(START, "chatbot")

# จาก chatbot → tools หรือ END (ขึ้นอยู่กับว่า LLM ต้องการเรียก tool หรือไม่)
# tools_condition เป็น built-in function ที่ช่วยตรวจสอบว่า LLM ตอบกลับมาพร้อม tool_calls หรือไม่
graph_builder.add_conditional_edges("chatbot", tools_condition)

# จาก tools → chatbot (หลังจาก tool ทำงานเสร็จ ส่งผลลัพธ์กลับให้ LLM วิเคราะห์ต่อ)
graph_builder.add_edge("tools", "chatbot")

# Compile graph
graph = graph_builder.compile()

# ---- 6. ทดสอบ ----
result = graph.invoke({
    "messages": [{"role": "user", "content": "อากาศที่กรุงเทพเป็นยังไงบ้าง ?"}]
})

# แสดงผลลัพธ์
for msg in result["messages"]:
    print(f"[{msg.type}] {msg.content}")
```

**Output:**
```
[human] อากาศที่กรุงเทพเป็นยังไงบ้าง ?
[ai]                              ← (AI ตัดสินใจเรียก tool, content ว่าง)
[tool] กรุงเทพฯ: อุณหภูมิ 35°C, แดดจัด, ความชื้น 70%
[ai] สภาพอากาศที่กรุงเทพฯ ตอนนี้ค่อนข้างร้อนครับ อุณหภูมิอยู่ที่ 35°C
     แดดจัดและมีความชื้นสูงถึง 70% แนะนำให้ดื่มน้ำเยอะๆ นะครับ
```

**อธิบาย Flow ที่เกิดขึ้น:**

```
START → chatbot (LLM คิดว่าต้องเรียก get_weather)
      → tools   (เรียก get_weather("bangkok"))
      → chatbot (LLM ได้ผลลัพธ์จาก tool แล้วสรุปคำตอบ)
      → END     (ไม่ต้องเรียก tool อีก จึงจบ)
```

### ตัวอย่างที่ 2: Graph แบบ Conditional Branching

ตัวอย่างที่ซับซ้อนขึ้น — Agent ที่ตัดสินใจว่าจะตอบเอง หรือส่งต่อให้ "ผู้เชี่ยวชาญ":

```python
import os
from dotenv import load_dotenv
from typing import Annotated, Literal
from typing_extensions import TypedDict

from langchain_google_genai import ChatGoogleGenerativeAI
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

load_dotenv()

# ---- State ----
class State(TypedDict):
    messages: Annotated[list, add_messages]
    category: str  # เก็บหมวดหมู่ของคำถาม

model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

# ---- Node 1: จำแนกหมวดหมู่คำถาม ----
def classify_question(state: State):
    """ให้ LLM จำแนกว่าคำถามเป็นหมวดอะไร"""
    classification_prompt = [
        {"role": "system", "content": """จำแนกคำถามต่อไปนี้เป็นหมวดหมู่ใดหมวดหมู่หนึ่ง:
        - "technical" สำหรับคำถามเกี่ยวกับการเขียนโปรแกรมหรือเทคโนโลยี
        - "general" สำหรับคำถามทั่วไป
        ตอบเพียงคำเดียว: technical หรือ general"""},
        state["messages"][-1]
    ]
    response = model.invoke(classification_prompt)
    category = response.content.strip().lower()

    # ทำให้แน่ใจว่าค่าเป็น technical หรือ general เท่านั้น
    if "technical" in category:
        category = "technical"
    else:
        category = "general"

    return {"category": category}

# ---- Node 2: ตอบคำถาม Technical ----
def technical_expert(state: State):
    """ผู้เชี่ยวชาญด้านเทคนิค"""
    messages = [
        {"role": "system", "content": "คุณคือผู้เชี่ยวชาญด้านการเขียนโปรแกรม ตอบอย่างละเอียดพร้อมตัวอย่าง code"},
        state["messages"][-1]
    ]
    response = model.invoke(messages)
    return {"messages": [response]}

# ---- Node 3: ตอบคำถาม General ----
def general_assistant(state: State):
    """ผู้ช่วยทั่วไป"""
    messages = [
        {"role": "system", "content": "คุณคือผู้ช่วยทั่วไปที่ตอบคำถามอย่างเป็นมิตรและกระชับ"},
        state["messages"][-1]
    ]
    response = model.invoke(messages)
    return {"messages": [response]}

# ---- Routing Function ----
def route_question(state: State) -> Literal["technical_expert", "general_assistant"]:
    if state["category"] == "technical":
        return "technical_expert"
    return "general_assistant"

# ---- สร้าง Graph ----
graph_builder = StateGraph(State)

graph_builder.add_node("classify", classify_question)
graph_builder.add_node("technical_expert", technical_expert)
graph_builder.add_node("general_assistant", general_assistant)

graph_builder.add_edge(START, "classify")
graph_builder.add_conditional_edges("classify", route_question)
graph_builder.add_edge("technical_expert", END)
graph_builder.add_edge("general_assistant", END)

graph = graph_builder.compile()

# ---- ทดสอบ ----
# คำถาม Technical
result1 = graph.invoke({
    "messages": [{"role": "user", "content": "อธิบาย async/await ใน Python หน่อย"}],
    "category": ""
})
print("=== Technical Question ===")
print(result1["messages"][-1].content)

# คำถาม General
result2 = graph.invoke({
    "messages": [{"role": "user", "content": "แนะนำร้านอาหารในกรุงเทพหน่อย"}],
    "category": ""
})
print("\n=== General Question ===")
print(result2["messages"][-1].content)
```

**Output:**
```
=== Technical Question ===
async/await ใน Python คือ syntax สำหรับเขียน Asynchronous Programming ครับ

ตัวอย่าง:
    import asyncio

    async def fetch_data():
        print("เริ่มดึงข้อมูล...")
        await asyncio.sleep(2)  # จำลองการรอ I/O
        print("ดึงข้อมูลเสร็จ!")
        return {"data": "Hello"}

    async def main():
        result = await fetch_data()
        print(result)

    asyncio.run(main())

หลักการคือ:
- async def ใช้ประกาศ coroutine function
- await ใช้รอผลลัพธ์จาก coroutine อื่น โดยไม่บล็อก event loop
- เหมาะกับงาน I/O-bound เช่น API calls, database queries

=== General Question ===
แนะนำร้านอาหารในกรุงเทพให้ครับ:
- ข้าวต้มปลา ตลาดพลู (อาหารเช้าสุดคลาสสิก)
- เจ๊ไฝ ที่ราชวงศ์ (ผัดไทยชื่อดัง)
- บ้านหญิง คาเฟ่ (บรรยากาศดี อาหารไทยรสชาติดั้งเดิม)
```

**Flow ของ Graph:**

```
START → classify → (ถ้า technical) → technical_expert → END
                  → (ถ้า general)   → general_assistant → END
```

### ตัวอย่างที่ 3: Human-in-the-Loop

LangGraph รองรับการ "หยุด" กลางทาง เพื่อให้มนุษย์ตรวจสอบก่อนดำเนินการต่อ:

```python
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.checkpoint.memory import MemorySaver

# ... (สร้าง State, Tools, Model เหมือนเดิม)

graph_builder = StateGraph(State)
graph_builder.add_node("chatbot", chatbot)
graph_builder.add_node("tools", ToolNode(tools=tools))

graph_builder.add_edge(START, "chatbot")
graph_builder.add_conditional_edges("chatbot", tools_condition)
graph_builder.add_edge("tools", "chatbot")

# เพิ่ม MemorySaver เป็น checkpointer
# และระบุ interrupt_before=["tools"] เพื่อหยุดก่อนเรียก tool
memory = MemorySaver()
graph = graph_builder.compile(
    checkpointer=memory,
    interrupt_before=["tools"]  # ← หยุดก่อนเรียก tools ทุกครั้ง
)

# ทดสอบ
config = {"configurable": {"thread_id": "1"}}

# รอบแรก: LLM ตัดสินใจจะเรียก tool แต่ graph หยุดก่อน
result = graph.invoke(
    {"messages": [{"role": "user", "content": "อากาศที่ภูเก็ตเป็นยังไง?"}]},
    config
)
print("Agent ต้องการเรียก tool:", result["messages"][-1].tool_calls)

# ณ จุดนี้ เราสามารถตรวจสอบว่า tool call ถูกต้องหรือไม่
# ถ้า OK → เรียก invoke ต่อ โดยส่ง None (ไม่เพิ่มข้อความใหม่)
user_approval = input("อนุมัติให้เรียก tool หรือไม่? (y/n): ")

if user_approval.lower() == "y":
    result = graph.invoke(None, config)  # ← ดำเนินการต่อจากจุดที่หยุดไว้
    print("คำตอบสุดท้าย:", result["messages"][-1].content)
else:
    print("ยกเลิกการเรียก tool")
```

**Output:**
```
Agent ต้องการเรียก tool: [{'name': 'get_weather', 'args': {'city': 'phuket'}}]
อนุมัติให้เรียก tool หรือไม่? (y/n): y
คำตอบสุดท้าย: สภาพอากาศที่ภูเก็ตตอนนี้ อุณหภูมิ 32°C มีฝนตกเป็นระยะ
              และความชื้นค่อนข้างสูงที่ 80% แนะนำพกร่มไปด้วยนะครับ
```

---

## 2. LangSmith

### LangSmith คืออะไร ?

LangSmith คือ **Platform สำหรับ Observability และ Evaluation** ของ LLM Applications ที่พัฒนาโดย LangChain team เพื่อช่วยให้เราสามารถ:

- **Tracing:** ดู trace ของทุกขั้นตอนที่เกิดขึ้นใน Agent ว่าแต่ละ step ส่ง prompt อะไร ได้คำตอบอะไรกลับมา ใช้เวลาและ token เท่าไหร่
- **Debugging:** เมื่อ Agent ตอบผิดหรือทำงานไม่ถูกต้อง สามารถดู trace ย้อนกลับเพื่อหาจุดที่ผิดพลาดได้
- **Evaluation:** สร้างชุดทดสอบ (Datasets) และรัน Evaluation เพื่อวัดคุณภาพของ Agent อย่างเป็นระบบ
- **Monitoring:** ดู Dashboard แบบ real-time เพื่อติดตาม performance, error rate, latency ของ Agent ใน production

### ตั้งค่า LangSmith

```bash
pip install -U langsmith
```

สร้าง API Key ได้ที่ [smith.langchain.com](https://smith.langchain.com) จากนั้นเพิ่มใน `.env`:

```env
LANGSMITH_API_KEY=lsv2_pt_xxxxx
LANGSMITH_TRACING=true
LANGSMITH_PROJECT=my-first-project
```

### ตัวอย่าง: เปิดใช้ Tracing อัตโนมัติ

เมื่อตั้งค่า environment variables ข้างต้นแล้ว LangChain จะส่ง trace ไปยัง LangSmith **โดยอัตโนมัติ** โดยไม่ต้องแก้ code:

```python
import os
from dotenv import load_dotenv
from langchain_google_genai import ChatGoogleGenerativeAI

load_dotenv()

# แค่ตั้งค่า env เท่านั้น code ส่วนอื่นไม่ต้องเปลี่ยนเลย
# LangSmith จะ trace ทุก invoke โดยอัตโนมัติ

model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")
response = model.invoke("อธิบาย RAG ให้หน่อย")
print(response.content)

# หลังจากรันเสร็จ ไปดู trace ได้ที่ smith.langchain.com
```

**สิ่งที่จะเห็นบน LangSmith Dashboard:**

```
📦 Trace: ChatGoogleGenerativeAI.invoke
├── ⏱️ Latency: 1.2s
├── 💰 Tokens: Input 12 | Output 156
├── 📥 Input:
│   └── messages: [HumanMessage("อธิบาย RAG ให้หน่อย")]
├── 📤 Output:
│   └── AIMessage("RAG คือ Retrieval-Augmented Generation...")
└── ✅ Status: Success
```

### ตัวอย่าง: Tracing แบบ Custom ด้วย @traceable

สำหรับ function ที่เราเขียนเอง (ไม่ใช่ LangChain) สามารถใช้ decorator `@traceable`:

```python
from langsmith import traceable
from langchain_google_genai import ChatGoogleGenerativeAI

model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")

@traceable(name="translate_and_summarize")
def translate_and_summarize(text: str, target_lang: str) -> dict:
    """แปลภาษาแล้วสรุป"""

    # Step 1: แปลภาษา
    translation = model.invoke(f"แปลข้อความนี้เป็น {target_lang}: {text}")

    # Step 2: สรุป
    summary = model.invoke(f"สรุปข้อความนี้ให้สั้นๆ ใน 1 ประโยค: {translation.content}")

    return {
        "original": text,
        "translation": translation.content,
        "summary": summary.content
    }

result = translate_and_summarize(
    text="LangChain เป็น framework สำหรับสร้าง AI application ที่ทำงานร่วมกับ LLM",
    target_lang="English"
)
print(result)
```

**Trace บน LangSmith:**

```
📦 Trace: translate_and_summarize
├── 📦 ChatGoogleGenerativeAI.invoke (แปลภาษา)
│   ├── ⏱️ 0.8s
│   └── 📤 "LangChain is a framework for building AI applications..."
├── 📦 ChatGoogleGenerativeAI.invoke (สรุป)
│   ├── ⏱️ 0.6s
│   └── 📤 "LangChain is an LLM-powered AI app framework."
├── ⏱️ Total: 1.4s
└── ✅ Success
```

### ตัวอย่าง: Evaluation ด้วย LangSmith

สร้าง Dataset และรัน Evaluation เพื่อวัดคุณภาพ:

```python
from langsmith import Client, evaluate
from langchain_google_genai import ChatGoogleGenerativeAI

client = Client()
model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")

# ---- 1. สร้าง Dataset ----
dataset = client.create_dataset("thai-qa-test", description="ชุดทดสอบ Q&A ภาษาไทย")

# เพิ่มตัวอย่างทดสอบ
examples = [
    {
        "inputs": {"question": "เมืองหลวงของประเทศไทยคืออะไร?"},
        "outputs": {"answer": "กรุงเทพมหานคร"}
    },
    {
        "inputs": {"question": "Python เป็นภาษาอะไร?"},
        "outputs": {"answer": "ภาษาโปรแกรมมิ่ง"}
    },
    {
        "inputs": {"question": "1 กิโลเมตร เท่ากับกี่เมตร?"},
        "outputs": {"answer": "1000 เมตร"}
    },
]

for ex in examples:
    client.create_example(
        inputs=ex["inputs"],
        outputs=ex["outputs"],
        dataset_id=dataset.id
    )

# ---- 2. กำหนด Target Function (ฟังก์ชันที่จะทดสอบ) ----
def my_qa_bot(inputs: dict) -> dict:
    response = model.invoke(inputs["question"])
    return {"answer": response.content}

# ---- 3. กำหนด Evaluator ----
def correctness_evaluator(run, example) -> dict:
    """ตรวจสอบว่าคำตอบมีคำหลักที่ถูกต้องหรือไม่"""
    predicted = run.outputs["answer"].lower()
    expected = example.outputs["answer"].lower()
    score = 1.0 if expected in predicted else 0.0
    return {"key": "correctness", "score": score}

# ---- 4. รัน Evaluation ----
results = evaluate(
    my_qa_bot,
    data="thai-qa-test",
    evaluators=[correctness_evaluator],
    experiment_prefix="gemini-flash-lite-test"
)
```

**Output บน LangSmith Dashboard:**

```
Experiment: gemini-flash-lite-test

| Question                           | Expected        | Predicted            | Correctness |
|------------------------------------|-----------------|----------------------|-------------|
| เมืองหลวงของประเทศไทยคืออะไร?     | กรุงเทพมหานคร    | กรุงเทพมหานครครับ     | ✅ 1.0      |
| Python เป็นภาษาอะไร?              | ภาษาโปรแกรมมิ่ง  | ภาษาโปรแกรมมิ่ง...   | ✅ 1.0      |
| 1 กิโลเมตร เท่ากับกี่เมตร?         | 1000 เมตร       | 1000 เมตรครับ        | ✅ 1.0      |

Average Correctness: 1.0
```

---

## 3. RAG — Retrieval-Augmented Generation

### RAG คืออะไร ?

RAG (Retrieval-Augmented Generation) คือเทคนิคที่ **เสริมความสามารถให้ LLM ด้วยการดึงข้อมูลจากแหล่งภายนอก** มาใช้ประกอบการตอบคำถาม แทนที่จะพึ่งพาแค่ความรู้ที่โมเดลเรียนรู้มาตอนฝึก (Training data) เพียงอย่างเดียว

**ทำไมต้อง RAG?**

- LLM มี Knowledge cutoff (ข้อมูลเก่า)
- LLM ไม่รู้ข้อมูลเฉพาะขององค์กร (เอกสารภายใน, คู่มือ, FAQ)
- LLM อาจ "หลอน" (Hallucinate) สร้างข้อมูลที่ไม่จริงขึ้นมา
- RAG ช่วยให้ LLM ตอบจากข้อมูลจริง โดยอ้างอิงแหล่งที่มาได้

### หลักการทำงานของ RAG

```
User Question
     │
     ▼
┌──────────┐    ┌───────────────┐    ┌──────────┐
│ 1. Embed │───▶│ 2. Retrieve   │───▶│ 3. Generate │
│ Question │    │ จาก VectorDB  │    │ คำตอบจาก LLM│
└──────────┘    └───────────────┘    └──────────┘
                                          │
                                          ▼
                                    Final Answer
                                  (พร้อม Context)
```

**3 ขั้นตอนหลัก:**

1. **Indexing (ทำครั้งเดียว):** นำเอกสาร → ตัดเป็นชิ้นเล็กๆ (Chunks) → แปลงเป็น Embeddings → เก็บใน Vector Store
2. **Retrieval:** เมื่อมีคำถาม → แปลงคำถามเป็น Embedding → ค้นหา Chunks ที่คล้ายกันมากที่สุดจาก Vector Store
3. **Generation:** นำ Chunks ที่ดึงมาได้ → ส่งเป็น Context ให้ LLM → LLM ตอบคำถามโดยอ้างอิงจาก Context

### ติดตั้ง Library สำหรับ RAG

```bash
pip install -U langchain[google-genai] langchain-chroma langchain-text-splitters
```

### ตัวอย่างที่ 1: RAG แบบพื้นฐาน — ถามจากเอกสาร

```python
import os
from dotenv import load_dotenv
from langchain_google_genai import ChatGoogleGenerativeAI, GoogleGenerativeAIEmbeddings
from langchain_chroma import Chroma
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_core.documents import Document

load_dotenv()

# ==== STEP 1: Indexing — เตรียมข้อมูลและเก็บใน Vector Store ====

# 1.1 สร้างเอกสารตัวอย่าง (ในงานจริงอาจโหลดจาก PDF, เว็บ, ฐานข้อมูล)
documents = [
    Document(
        page_content="""LangChain คือ Open Source Framework สำหรับสร้าง AI Application
        ที่ทำงานร่วมกับ LLM (Large Language Models) ถูกสร้างขึ้นในปี 2022 โดย Harrison Chase
        มีจุดเด่นคือ รองรับการเชื่อมต่อกับ LLM หลายเจ้า, มีระบบ Agent สำเร็จรูป,
        และมี Ecosystem ที่ครบถ้วนทั้ง LangGraph และ LangSmith""",
        metadata={"source": "langchain-docs", "topic": "overview"}
    ),
    Document(
        page_content="""LangGraph เป็น library ที่สร้างบน LangChain สำหรับควบคุม
        Flow การทำงานของ Agent อย่างละเอียด ใช้แนวคิดของ Graph ที่มี Nodes และ Edges
        รองรับ Human-in-the-loop, Conditional branching และ Multi-agent systems
        เหมาะกับงานที่ต้องการ workflow ที่ซับซ้อนกว่า create_agent ปกติ""",
        metadata={"source": "langgraph-docs", "topic": "langgraph"}
    ),
    Document(
        page_content="""LangSmith คือ Platform สำหรับ Observability และ Evaluation
        ของ LLM Applications ช่วยให้ดู Trace การทำงานของ Agent ทุกขั้นตอน
        สามารถสร้าง Dataset สำหรับทดสอบ และรัน Evaluation วัดคุณภาพได้
        รองรับทั้ง Tracing อัตโนมัติ และ Custom tracing ด้วย @traceable""",
        metadata={"source": "langsmith-docs", "topic": "langsmith"}
    ),
    Document(
        page_content="""RAG (Retrieval-Augmented Generation) คือเทคนิคที่เสริม LLM
        ด้วยการดึงข้อมูลจากแหล่งภายนอก มี 3 ขั้นตอนหลัก:
        1. Indexing: ตัดเอกสารเป็น Chunks แล้วเก็บเป็น Embeddings ใน Vector Store
        2. Retrieval: ค้นหา Chunks ที่เกี่ยวข้องกับคำถาม
        3. Generation: ส่ง Chunks เป็น Context ให้ LLM สร้างคำตอบ""",
        metadata={"source": "rag-guide", "topic": "rag"}
    ),
    Document(
        page_content="""Embedding คือการแปลงข้อความให้เป็น Vector ตัวเลข
        ที่สามารถนำมาเปรียบเทียบความคล้ายกันได้ (Similarity Search)
        ใน LangChain สามารถใช้ Embedding จากหลายเจ้าเช่น
        Google (text-embedding-004), OpenAI (text-embedding-3-small)
        ค่า Embedding จะถูกเก็บใน Vector Store เช่น Chroma, FAISS, Pinecone""",
        metadata={"source": "embedding-guide", "topic": "embedding"}
    ),
]

# 1.2 ตัดเอกสารเป็นชิ้นเล็กๆ (Chunking)
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=300,       # ขนาดสูงสุดของแต่ละ chunk (ตัวอักษร)
    chunk_overlap=50,     # จำนวนตัวอักษรที่ซ้อนทับกันระหว่าง chunk
    separators=["\n\n", "\n", ".", " "]  # ลำดับการตัด (พยายามตัดที่ย่อหน้า > บรรทัด > ประโยค > คำ)
)

chunks = text_splitter.split_documents(documents)
print(f"ตัดเอกสารได้ทั้งหมด {len(chunks)} chunks")

# 1.3 สร้าง Embeddings และเก็บใน Vector Store (Chroma)
embeddings = GoogleGenerativeAIEmbeddings(model="models/text-embedding-004")

vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings,
    collection_name="langchain-tutorial",
    persist_directory="./chroma_db"  # เก็บลงดิสก์ เพื่อไม่ต้องสร้างใหม่ทุกครั้ง
)

print("สร้าง Vector Store เรียบร้อย!")


# ==== STEP 2: Retrieval + Generation ====

# 2.1 สร้าง Retriever จาก Vector Store
retriever = vectorstore.as_retriever(
    search_type="similarity",  # ค้นหาด้วยความคล้าย
    search_kwargs={"k": 3}     # ดึงมา 3 chunks ที่เกี่ยวข้องที่สุด
)

# 2.2 สร้าง LLM
model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

# 2.3 สร้าง RAG function
def rag_answer(question: str) -> dict:
    """ถามคำถามแบบ RAG: ดึงข้อมูล → สร้างคำตอบ"""

    # Retrieve: ดึง chunks ที่เกี่ยวข้อง
    relevant_chunks = retriever.invoke(question)

    # สร้าง context จาก chunks
    context = "\n\n---\n\n".join([chunk.page_content for chunk in relevant_chunks])

    # สร้าง sources สำหรับอ้างอิง
    sources = list(set([chunk.metadata.get("source", "unknown") for chunk in relevant_chunks]))

    # Generate: ส่ง context + คำถาม ให้ LLM
    prompt = [
        {"role": "system", "content": f"""คุณคือผู้ช่วย AI ที่ตอบคำถามโดยอ้างอิงจากข้อมูลที่ให้มาเท่านั้น
ถ้าไม่มีข้อมูลเพียงพอ ให้บอกตรงๆ ว่าไม่มีข้อมูลในเอกสาร

ข้อมูลอ้างอิง:
{context}"""},
        {"role": "user", "content": question}
    ]

    response = model.invoke(prompt)

    return {
        "question": question,
        "answer": response.content,
        "sources": sources,
        "num_chunks_used": len(relevant_chunks)
    }


# ==== STEP 3: ทดสอบ ====

# คำถามที่ 1
result1 = rag_answer("LangGraph คืออะไร และแตกต่างจาก LangChain อย่างไร?")
print(f"Q: {result1['question']}")
print(f"A: {result1['answer']}")
print(f"Sources: {result1['sources']}")
print(f"Chunks used: {result1['num_chunks_used']}")
print()

# คำถามที่ 2
result2 = rag_answer("Embedding ใช้ทำอะไรใน RAG?")
print(f"Q: {result2['question']}")
print(f"A: {result2['answer']}")
print(f"Sources: {result2['sources']}")
print(f"Chunks used: {result2['num_chunks_used']}")
print()

# คำถามที่ 3: ทดสอบกรณีไม่มีข้อมูล
result3 = rag_answer("วิธีทำ Pad Thai เป็นยังไง?")
print(f"Q: {result3['question']}")
print(f"A: {result3['answer']}")
print(f"Sources: {result3['sources']}")
```

**Output:**
```
ตัดเอกสารได้ทั้งหมด 8 chunks
สร้าง Vector Store เรียบร้อย!

Q: LangGraph คืออะไร และแตกต่างจาก LangChain อย่างไร?
A: LangGraph เป็น library ที่สร้างบน LangChain สำหรับควบคุม Flow การทำงานของ
   Agent อย่างละเอียด โดยใช้แนวคิดของ Graph ที่มี Nodes และ Edges

   ความแตกต่างหลักคือ LangChain เป็น Framework ภาพรวมสำหรับสร้าง AI Application
   ที่มี Agent สำเร็จรูป ส่วน LangGraph เจาะลึกลงไปที่การควบคุม workflow
   ที่ซับซ้อนกว่า เช่น Conditional branching, Human-in-the-loop
   และ Multi-agent systems

Sources: ['langgraph-docs', 'langchain-docs']
Chunks used: 3

Q: Embedding ใช้ทำอะไรใน RAG?
A: ใน RAG, Embedding ทำหน้าที่แปลงข้อความ (ทั้งเอกสารและคำถาม) ให้เป็น
   Vector ตัวเลข เพื่อนำมาเปรียบเทียบความคล้ายกันได้ (Similarity Search)

   ในขั้นตอน Indexing: เอกสารที่ตัดเป็น Chunks จะถูกแปลงเป็น Embeddings
   แล้วเก็บใน Vector Store เช่น Chroma, FAISS, Pinecone

   ในขั้นตอน Retrieval: คำถามจะถูกแปลงเป็น Embedding เช่นกัน แล้วค้นหา
   Chunks ที่มี vector ใกล้เคียงกัน (คล้ายกันมากที่สุด)

Sources: ['embedding-guide', 'rag-guide']
Chunks used: 3

Q: วิธีทำ Pad Thai เป็นยังไง?
A: ขออภัยครับ ไม่มีข้อมูลเกี่ยวกับวิธีทำ Pad Thai ในเอกสารที่มีอยู่
   เอกสารที่มีเป็นเนื้อหาเกี่ยวกับ LangChain, LangGraph, LangSmith
   และ RAG เท่านั้นครับ

Sources: ['langchain-docs', 'rag-guide']
Chunks used: 3
```

### ตัวอย่างที่ 2: โหลดเอกสารจาก PDF

ในงานจริง เรามักจะโหลดเอกสารจากไฟล์ PDF:

```bash
pip install pypdf
```

```python
from langchain_community.document_loaders import PyPDFLoader

# โหลด PDF
loader = PyPDFLoader("company-handbook.pdf")
documents = loader.load()

print(f"โหลดได้ {len(documents)} หน้า")
print(f"หน้าแรก: {documents[0].page_content[:200]}...")
print(f"Metadata: {documents[0].metadata}")
# Output: {'source': 'company-handbook.pdf', 'page': 0}

# จากนั้นนำไป split → embed → store เหมือนเดิม
chunks = text_splitter.split_documents(documents)
vectorstore = Chroma.from_documents(documents=chunks, embedding=embeddings)
```

### ตัวอย่างที่ 3: โหลดจากเว็บไซต์

```bash
pip install beautifulsoup4
```

```python
from langchain_community.document_loaders import WebBaseLoader

loader = WebBaseLoader("https://mikelopster.dev/posts/langchain-101")
documents = loader.load()

print(f"โหลดได้ {len(documents)} documents")
print(f"เนื้อหา: {documents[0].page_content[:300]}...")
```

---

## 4. รวมทุกอย่างเข้าด้วยกัน — RAG Agent with LangGraph

นี่คือตัวอย่างสุดท้ายที่รวม LangGraph + RAG + Tools เข้าด้วยกัน:

```python
import os
from dotenv import load_dotenv
from typing import Annotated, Literal
from typing_extensions import TypedDict

from langchain_google_genai import ChatGoogleGenerativeAI, GoogleGenerativeAIEmbeddings
from langchain_chroma import Chroma
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_core.documents import Document
from langchain.tools import tool
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

load_dotenv()

# ==== Setup Vector Store (เหมือนตัวอย่าง RAG ด้านบน) ====
documents = [
    Document(page_content="บริษัท ABC มีพนักงาน 500 คน ก่อตั้งปี 2010 สำนักงานใหญ่อยู่ที่กรุงเทพฯ",
             metadata={"source": "company-info"}),
    Document(page_content="นโยบายลาพักร้อน: พนักงานมีสิทธิ์ลาพักร้อน 15 วันต่อปี ลาป่วย 30 วันต่อปี",
             metadata={"source": "hr-policy"}),
    Document(page_content="สวัสดิการ: ประกันสุขภาพกลุ่ม, ค่าอาหารกลางวัน 100 บาท/วัน, ค่าเดินทาง 2000 บาท/เดือน",
             metadata={"source": "hr-policy"}),
]

embeddings = GoogleGenerativeAIEmbeddings(model="models/text-embedding-004")
text_splitter = RecursiveCharacterTextSplitter(chunk_size=200, chunk_overlap=30)
chunks = text_splitter.split_documents(documents)
vectorstore = Chroma.from_documents(documents=chunks, embedding=embeddings)
retriever = vectorstore.as_retriever(search_kwargs={"k": 2})

# ==== สร้าง Tools ====
@tool
def search_company_docs(query: str) -> str:
    """ค้นหาข้อมูลจากเอกสารภายในบริษัท เช่น นโยบาย HR, สวัสดิการ, ข้อมูลบริษัท"""
    results = retriever.invoke(query)
    if not results:
        return "ไม่พบข้อมูลที่เกี่ยวข้องในเอกสารบริษัท"
    context = "\n".join([doc.page_content for doc in results])
    sources = [doc.metadata.get("source", "unknown") for doc in results]
    return f"ข้อมูลที่พบ:\n{context}\n\n(แหล่งที่มา: {', '.join(set(sources))})"

@tool
def calculate(expression: str) -> str:
    """คำนวณนิพจน์ทางคณิตศาสตร์ เช่น '15 * 12' หรือ '2000 + 100 * 22'"""
    try:
        result = eval(expression)
        return f"ผลลัพธ์ของ {expression} = {result}"
    except Exception as e:
        return f"ไม่สามารถคำนวณได้: {e}"

tools = [search_company_docs, calculate]

# ==== สร้าง Graph ====
class State(TypedDict):
    messages: Annotated[list, add_messages]

model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)
model_with_tools = model.bind_tools(tools)

def agent_node(state: State):
    system_msg = {"role": "system", "content": """คุณคือ HR Assistant ของบริษัท ABC
ใช้เครื่องมือ search_company_docs เพื่อค้นหาข้อมูลจากเอกสารบริษัท
ใช้เครื่องมือ calculate เมื่อต้องคำนวณตัวเลข
ตอบเป็นภาษาไทย อ้างอิงจากข้อมูลที่ค้นหาได้เท่านั้น"""}

    messages = [system_msg] + state["messages"]
    response = model_with_tools.invoke(messages)
    return {"messages": [response]}

graph_builder = StateGraph(State)
graph_builder.add_node("agent", agent_node)
graph_builder.add_node("tools", ToolNode(tools=tools))

graph_builder.add_edge(START, "agent")
graph_builder.add_conditional_edges("agent", tools_condition)
graph_builder.add_edge("tools", "agent")

graph = graph_builder.compile()

# ==== ทดสอบ ====
# คำถาม 1: ค้นหาจากเอกสาร
result = graph.invoke({
    "messages": [{"role": "user", "content": "ลาพักร้อนได้กี่วันต่อปี?"}]
})
print("Q: ลาพักร้อนได้กี่วันต่อปี?")
print(f"A: {result['messages'][-1].content}")
print()

# คำถาม 2: ค้นหา + คำนวณ
result = graph.invoke({
    "messages": [{"role": "user", "content": "ถ้าทำงาน 22 วันต่อเดือน ค่าอาหารรวมต่อเดือนเท่าไหร่?"}]
})
print("Q: ถ้าทำงาน 22 วันต่อเดือน ค่าอาหารรวมต่อเดือนเท่าไหร่?")
print(f"A: {result['messages'][-1].content}")
```

**Output:**
```
Q: ลาพักร้อนได้กี่วันต่อปี?
A: ตามนโยบายของบริษัท ABC พนักงานมีสิทธิ์ลาพักร้อน 15 วันต่อปี
   และลาป่วยได้ 30 วันต่อปีครับ (อ้างอิงจาก: hr-policy)

Q: ถ้าทำงาน 22 วันต่อเดือน ค่าอาหารรวมต่อเดือนเท่าไหร่?
A: จากข้อมูลสวัสดิการ บริษัทให้ค่าอาหารกลางวัน 100 บาท/วัน
   ถ้าทำงาน 22 วัน/เดือน ค่าอาหารรวม = 100 × 22 = 2,200 บาท/เดือนครับ
```

---

## สรุปภาพรวม Ecosystem

```
┌─────────────────────────────────────────────────────┐
│                   LangSmith                         │
│         (Tracing, Evaluation, Monitoring)           │
├─────────────────────────────────────────────────────┤
│                                                     │
│   ┌───────────────────────────────────────────┐     │
│   │              LangGraph                    │     │
│   │    (Complex Flows, Multi-Agent,           │     │
│   │     Human-in-the-loop)                    │     │
│   ├───────────────────────────────────────────┤     │
│   │              LangChain                    │     │
│   │    (Models, Messages, Tools, Agents,      │     │
│   │     RAG, Structured Output)               │     │
│   └───────────────────────────────────────────┘     │
│                                                     │
│   ┌──────────┐  ┌──────────┐  ┌──────────────┐     │
│   │  LLMs    │  │  Vector  │  │  External    │     │
│   │ (Gemini, │  │  Stores  │  │  Tools       │     │
│   │  GPT,    │  │ (Chroma, │  │ (Search,     │     │
│   │  Claude) │  │  FAISS)  │  │  DB, API)    │     │
│   └──────────┘  └──────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────┘
```

### แนวทางเลือกใช้

- **แค่อยากคุยกับ LLM + Tools ง่ายๆ** → ใช้ `LangChain` + `create_agent`
- **ต้องการ Flow ซับซ้อน / Multi-agent** → ใช้ `LangGraph`
- **ต้องการให้ LLM ตอบจากข้อมูลเฉพาะ** → ทำ `RAG`
- **ต้องการ Debug / วัดคุณภาพ** → ใช้ `LangSmith`
- **Production จริงจัง** → ใช้ทั้งหมดรวมกัน

---

## Next Steps แนะนำ

1. **Advanced RAG:** เทคนิค Hybrid Search (keyword + semantic), Re-ranking, Multi-query retrieval
2. **Multi-Agent Systems:** สร้าง Agent หลายตัวทำงานร่วมกันผ่าน LangGraph
3. **Deployment:** Deploy Agent ขึ้น Cloud ด้วย LangServe หรือ LangGraph Cloud
4. **Advanced Evaluation:** สร้าง Custom evaluator ที่ใช้ LLM เป็น judge (LLM-as-Judge)
