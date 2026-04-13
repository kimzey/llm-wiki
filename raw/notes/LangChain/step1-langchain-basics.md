# Step 1: LangChain พื้นฐาน
## Model, Messages, Prompt Templates — สอนตั้งแต่ศูนย์

> ยังไม่ต้องรู้ AI มาก่อน เริ่มจากติดตั้ง → คุยกับ AI → ส่งรูป → กำหนด output ได้

---

## 1.1 เตรียมตัว

### ติดตั้ง Python (ถ้ายังไม่มี)

```bash
# ตรวจว่ามี Python ไหม
python --version
# ถ้าไม่มี → ดาวน์โหลดที่ python.org (แนะนำ 3.11+)
```

### สร้าง Project

```bash
# สร้าง folder
mkdir my-ai-project
cd my-ai-project

# สร้าง virtual environment (แยก library ไม่ปนกับ project อื่น)
python -m venv venv

# เปิดใช้ virtual environment
# Windows:
venv\Scripts\activate
# Mac/Linux:
source venv/bin/activate

# ติดตั้ง LangChain + Gemini
pip install "langchain[google-genai]" python-dotenv
```

### ไปเอา API Key

1. ไปที่ [aistudio.google.com/apikey](https://aistudio.google.com/apikey)
2. กด "Create API Key"
3. Copy key มาเก็บ

### สร้างไฟล์ .env

```env
# ไฟล์ .env (อยู่ใน root ของ project)
GOOGLE_API_KEY=AIzaSyxxxxxxxxxxxxxxxxxxxxxxxxxx
```

> ⚠️ ห้าม push .env ขึ้น GitHub → เพิ่ม `.env` ใน `.gitignore`

---

## 1.2 Hello LLM — คุยกับ AI ครั้งแรก

### Code แรกของเรา

```python
# file: 01_hello.py

# ---- Import ----
import os                          # ใช้จัดการ environment variables
from dotenv import load_dotenv     # ใช้อ่านไฟล์ .env
from langchain_google_genai import ChatGoogleGenerativeAI  # class สำหรับ Gemini

# ---- โหลด API Key จาก .env ----
load_dotenv()
# load_dotenv() จะอ่านไฟล์ .env แล้วใส่ค่าเข้า os.environ ให้
# ทำให้ GOOGLE_API_KEY พร้อมใช้โดยไม่ต้อง hardcode

# ---- สร้าง Model ----
model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")
#       ▲                              ▲
#       │                              │
#       class จาก LangChain            ชื่อ model ที่จะใช้
#       ทำหน้าที่เชื่อมต่อ Gemini       (มีหลายตัว เช่น gemini-2.5-pro)

# ---- ส่งคำถาม ----
response = model.invoke("สวัสดีครับ คุณคือใคร?")
#                 ▲           ▲
#                 │           │
#           method สำหรับ    prompt (คำสั่ง/คำถาม)
#           ส่ง prompt       ที่ส่งให้ AI
#           ไปยัง AI

# ---- ดูผลลัพธ์ ----
print(response)          # ได้ AIMessage object ทั้งก้อน
print(response.content)  # ได้แค่เนื้อหาคำตอบ (string)
print(type(response))    # <class 'langchain_core.messages.ai.AIMessage'>
```

### รัน

```bash
python 01_hello.py
```

### Output

```
สวัสดีครับ! ผมคือ AI ผู้ช่วยจาก Google ครับ มีอะไรให้ช่วยไหมครับ?
```

### อธิบาย response object

```python
response = model.invoke("สวัสดี")

# response เป็น AIMessage object ประกอบด้วย:
response.content              # "สวัสดีครับ! ..." (คำตอบ)
response.response_metadata    # {'model_name': '...', 'usage_metadata': {...}}
response.id                   # "run-xxxxx" (ID ของการเรียกครั้งนี้)
response.tool_calls           # [] (ว่างเปล่า ถ้า AI ไม่ได้เรียก tool)
response.usage_metadata       # {'input_tokens': 5, 'output_tokens': 20, ...}
```

---

## 1.3 Model Parameters — ปรับพฤติกรรม AI

```python
# file: 02_parameters.py

from dotenv import load_dotenv
from langchain_google_genai import ChatGoogleGenerativeAI

load_dotenv()

model = ChatGoogleGenerativeAI(
    model="gemini-2.5-flash-lite",

    temperature=0.7,
    # temperature คืออะไร?
    # = ความ "สุ่ม" ในการตอบ
    # 0.0 = ตอบเหมือนเดิมทุกครั้ง (เหมาะ: ตอบคำถาม, คำนวณ)
    # 0.5 = balance
    # 1.0 = สร้างสรรค์ สุ่มมาก (เหมาะ: เขียนเรื่อง, brainstorm)
    #
    # ลองเปลี่ยนเป็น 0 แล้วถามซ้ำ 3 รอบ → ได้คำตอบเหมือนกัน
    # ลองเปลี่ยนเป็น 1 แล้วถามซ้ำ 3 รอบ → ได้คำตอบต่างกัน

    max_output_tokens=1024,
    # max_output_tokens คืออะไร?
    # = จำนวน token สูงสุดที่ AI จะสร้าง
    # 1 token ≈ 1 คำภาษาอังกฤษ / 1-2 ตัวอักษรภาษาไทย
    # ถ้าตั้งน้อย → คำตอบจะถูกตัดสั้น
    # ถ้าตั้งเยอะ → AI ตอบยาวได้ (แต่ไม่ได้แปลว่าจะตอบยาวเสมอ)

    top_p=0.95,
    # top_p (Nucleus Sampling) คืออะไร?
    # = พิจารณาเฉพาะ token ที่ความน่าจะเป็นรวมกันถึง p
    # 0.1 = เลือกจากตัวเลือกน้อย → คำตอบแคบ
    # 0.95 = เลือกจากตัวเลือกเยอะ → คำตอบหลากหลาย
    # ปกติไม่ต้องปรับ ใช้ค่า default ได้

    timeout=30,
    # timeout คืออะไร?
    # = รอ AI ตอบสูงสุดกี่วินาที ถ้าเกินจะ error
    # ป้องกัน request ค้างนานเกินไป

    max_retries=2,
    # max_retries คืออะไร?
    # = ถ้า API error (เช่น network timeout) ลองใหม่กี่ครั้ง
)

# ลองเปรียบเทียบ temperature
print("=== temperature=0 (ตอบตรงๆ) ===")
model_strict = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)
for i in range(3):
    r = model_strict.invoke("บอกเลข 1-5 มาสุ่มๆ 1 ตัว")
    print(f"  รอบ {i+1}: {r.content}")

print("\n=== temperature=1 (สร้างสรรค์) ===")
model_creative = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=1)
for i in range(3):
    r = model_creative.invoke("บอกเลข 1-5 มาสุ่มๆ 1 ตัว")
    print(f"  รอบ {i+1}: {r.content}")
```

**Output:**
```
=== temperature=0 (ตอบตรงๆ) ===
  รอบ 1: 3
  รอบ 2: 3
  รอบ 3: 3        ← เหมือนกันทุกรอบ!

=== temperature=1 (สร้างสรรค์) ===
  รอบ 1: 4
  รอบ 2: 2
  รอบ 3: 5        ← ต่างกันทุกรอบ!
```

---

## 1.4 วิธีเรียก Model — 4 แบบ

```python
# file: 03_invoke_methods.py

from dotenv import load_dotenv
from langchain_google_genai import ChatGoogleGenerativeAI

load_dotenv()
model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")

# ==============================
# วิธีที่ 1: invoke() — ได้คำตอบทั้งหมดทีเดียว
# ==============================
response = model.invoke("เมืองหลวงของไทยคืออะไร")
print(response.content)
# Output: "กรุงเทพมหานคร"
# ใช้เมื่อ: ต้องการคำตอบทั้งหมดก่อนแสดงผล


# ==============================
# วิธีที่ 2: stream() — ได้คำตอบทีละ chunk (เหมือน ChatGPT พิมพ์ทีละคำ)
# ==============================
print("\nStreaming: ", end="")
for chunk in model.stream("เล่านิทานสั้นๆ 2 ประโยค"):
    print(chunk.content, end="", flush=True)
    # chunk.content = ข้อความทีละส่วน
    # flush=True = แสดงผลทันที ไม่รอ buffer
print()
# Output ทีละคำ: "กาล..." "ครั้ง..." "หนึ่ง..." "นาน..." "มา..." "แล้ว..."
# ใช้เมื่อ: ทำ chatbot ให้ user เห็นคำตอบทีละคำ (UX ดีกว่า)


# ==============================
# วิธีที่ 3: batch() — ส่งหลาย prompt พร้อมกัน
# ==============================
responses = model.batch([
    "เมืองหลวงของไทย",
    "เมืองหลวงของญี่ปุ่น",
    "เมืองหลวงของฝรั่งเศส",
])
for r in responses:
    print(r.content)
# Output:
#   กรุงเทพมหานคร
#   โตเกียว
#   ปารีส
# ใช้เมื่อ: ต้องถามหลายคำถาม ประหยัดเวลากว่า invoke ทีละตัว


# ==============================
# วิธีที่ 4: ainvoke() — Async (ไม่บล็อก)
# ==============================
import asyncio

async def ask_async():
    response = await model.ainvoke("สวัสดี")
    #                 ▲
    #           await = รอผลลัพธ์แบบ async
    #           ระหว่างรอ โปรแกรมทำอย่างอื่นได้
    print(response.content)

asyncio.run(ask_async())
# ใช้เมื่อ: ทำ web server (FastAPI) ที่ต้อง handle หลาย request พร้อมกัน
```

---

## 1.5 Messages — ระบบสื่อสารกับ AI

### ทำไมต้องมี Messages?

```
ถ้าใช้แค่ text ธรรมดา:
  model.invoke("สวัสดี")
  → AI ไม่รู้ว่าเราเป็นใคร ไม่มีบริบท ไม่มีกฎ

ถ้าใช้ Messages:
  SystemMessage: "คุณคือหมอ ตอบเรื่องสุขภาพเท่านั้น"
  HumanMessage: "ปวดหัวทำยังไงดี"
  → AI รู้บทบาท รู้กฎ ตอบได้ตรงประเด็น
```

### ประเภท Messages ทั้งหมด

```python
# file: 04_messages.py

from dotenv import load_dotenv
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain.messages import (
    SystemMessage,    # กำหนดบทบาท/กฎให้ AI
    HumanMessage,     # ข้อความจากผู้ใช้
    AIMessage,        # คำตอบจาก AI (ใช้เมื่อจำลองประวัติ)
    ToolMessage,      # ผลลัพธ์จาก Tool (จะเรียนใน Step 2)
)

load_dotenv()
model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")
```

### SystemMessage — "บุคลิกภาพ" ของ AI

```python
# SystemMessage = บอก AI ว่า "คุณคือใคร ทำอะไร มีกฎอะไร"
# ใส่ตอนเริ่มบทสนทนา ก่อน HumanMessage เสมอ

messages = [
    SystemMessage(content="คุณคือเชฟอาหารไทยชื่อ 'เชฟต้น' ตอบเฉพาะเรื่องอาหารไทย ถ้าถามเรื่องอื่นให้ปฏิเสธ"),
    #            ▲
    #       content = เนื้อหาของ message

    HumanMessage(content="สอนทำต้มยำกุ้งหน่อย"),
]

response = model.invoke(messages)
print(response.content)
# Output: "สวัสดีครับ ผม เชฟต้น ต้มยำกุ้งทำง่ายมาก..."

# ลองถามนอกเรื่อง
messages2 = [
    SystemMessage(content="คุณคือเชฟอาหารไทยชื่อ 'เชฟต้น' ตอบเฉพาะเรื่องอาหารไทย ถ้าถามเรื่องอื่นให้ปฏิเสธ"),
    HumanMessage(content="สอนเขียน Python หน่อย"),
]
response2 = model.invoke(messages2)
print(response2.content)
# Output: "ขอโทษครับ ผมเชฟต้น ตอบได้เฉพาะเรื่องอาหารไทยครับ"
```

### Chat History — สร้างประวัติบทสนทนา

```python
# AI ไม่มีความจำ! ถ้าไม่ส่ง history ไป → AI จะลืมทุกอย่าง
# วิธีแก้: ส่ง messages ทั้งหมดไปทุกครั้ง

messages = [
    SystemMessage(content="คุณคือผู้ช่วย AI ตอบสั้นๆ"),

    # รอบที่ 1 (ประวัติ)
    HumanMessage(content="ผมชื่อ สมชาย ครับ"),
    AIMessage(content="สวัสดีครับ คุณสมชาย!"),
    #  ▲
    #  AIMessage = จำลองคำตอบเก่าของ AI
    #  ใส่เพื่อให้ AI รู้ว่าเคยตอบอะไรไป

    # รอบที่ 2 (ประวัติ)
    HumanMessage(content="ผมชอบกินส้มตำ"),
    AIMessage(content="ส้มตำอร่อยดีครับ!"),

    # รอบที่ 3 (คำถามปัจจุบัน)
    HumanMessage(content="ผมชื่ออะไร และชอบกินอะไร?"),
]

response = model.invoke(messages)
print(response.content)
# Output: "คุณชื่อ สมชาย และชอบกินส้มตำ ครับ"
# → AI จำได้! เพราะเราส่ง history ไปด้วย
```

### Dictionary Format — วิธีเขียนแบบสั้น

```python
# แทนที่จะใช้ class สามารถเขียนแบบ dict ได้ (ผลลัพธ์เหมือนกัน)

messages = [
    {"role": "system",    "content": "คุณคือผู้ช่วย AI"},
    {"role": "user",      "content": "สวัสดี"},
    {"role": "assistant", "content": "สวัสดีครับ!"},   # = AIMessage
    {"role": "user",      "content": "ขอบคุณ"},
]

# mapping:
# "system"    = SystemMessage
# "user"      = HumanMessage
# "assistant" = AIMessage
# "tool"      = ToolMessage

response = model.invoke(messages)
print(response.content)

# 💡 dict format นิยมกว่า เพราะ:
# - อ่านง่าย
# - แยกเป็นไฟล์ JSON ได้
# - ไม่ต้อง import class
```

### ส่งรูปภาพ (Multimodal)

```python
# file: 05_image.py

import base64
from dotenv import load_dotenv
from langchain_google_genai import ChatGoogleGenerativeAI

load_dotenv()
model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")

# ---- วิธีที่ 1: ส่งรูปจากไฟล์ (Base64) ----
def encode_image(path):
    """อ่านไฟล์รูปแล้วแปลงเป็น base64 string"""
    with open(path, "rb") as f:        # rb = read binary
        return base64.b64encode(f.read()).decode("utf-8")
        #      ▲                         ▲
        #      แปลง binary → base64      แปลง bytes → string

image_b64 = encode_image("photo.jpg")

message = {
    "role": "user",
    "content": [
        # content เป็น list ได้! (ส่งได้หลายอย่างพร้อมกัน)
        {"type": "text", "text": "อธิบายภาพนี้เป็นภาษาไทย"},
        {
            "type": "image",
            "base64": image_b64,          # รูปที่แปลงเป็น base64
            "mime_type": "image/jpeg",    # ประเภทไฟล์: image/jpeg, image/png
        },
    ]
}

response = model.invoke([message])
print(response.content)
# Output: "ในภาพเป็นแมวสีส้มนั่งอยู่บนโซฟา..."


# ---- วิธีที่ 2: ส่งรูปจาก URL ----
message_url = {
    "role": "user",
    "content": [
        {"type": "text", "text": "อธิบายภาพนี้"},
        {
            "type": "image_url",
            "image_url": {"url": "https://example.com/cat.jpg"}
        },
    ]
}

response = model.invoke([message_url])
print(response.content)
```

---

## 1.6 Prompt Templates — สร้าง Prompt แบบมี Template

### ปัญหาของ Prompt แบบ hardcode

```python
# ❌ ไม่ดี — hardcode prompt
prompt1 = "แปลเป็นภาษาอังกฤษ: สวัสดี"
prompt2 = "แปลเป็นภาษาอังกฤษ: ขอบคุณ"
prompt3 = "แปลเป็นภาษาญี่ปุ่น: สวัสดี"
# → ถ้ามี 100 คำ ต้องเขียน 100 บรรทัด!

# ✅ ดีกว่า — ใช้ template
# "แปลเป็นภาษา {language}: {text}"
# → ใส่ค่าตัวแปรทีหลังได้
```

### ChatPromptTemplate

```python
# file: 06_templates.py

from dotenv import load_dotenv
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain_core.prompts import ChatPromptTemplate

load_dotenv()
model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")

# ---- สร้าง Template ----
prompt = ChatPromptTemplate.from_messages([
    # from_messages() รับ list ของ tuple (role, template)
    ("system", "คุณคือนักแปลมืออาชีพ แปลได้ทุกภาษา"),
    ("human",  "แปลเป็นภาษา {language}: {text}"),
    #                       ▲             ▲
    #                  ตัวแปร          ตัวแปร
    #              (ใส่ค่าทีหลัง)   (ใส่ค่าทีหลัง)
])

# ---- ใส่ค่าตัวแปร ----
formatted = prompt.invoke({"language": "English", "text": "สวัสดีครับ"})
#                          ▲
#                     dict ของตัวแปร
#                     key ต้องตรงกับ {ชื่อ} ใน template

print(formatted.messages)
# Output:
# [SystemMessage(content='คุณคือนักแปลมืออาชีพ แปลได้ทุกภาษา'),
#  HumanMessage(content='แปลเป็นภาษา English: สวัสดีครับ')]

# ---- ส่งให้ Model ----
response = model.invoke(formatted)
print(response.content)
# Output: "Hello"

# ---- ใช้ซ้ำกับค่าต่าง ----
r2 = model.invoke(prompt.invoke({"language": "Japanese", "text": "ขอบคุณ"}))
print(r2.content)  # "ありがとう"

r3 = model.invoke(prompt.invoke({"language": "Korean", "text": "สวัสดี"}))
print(r3.content)  # "안녕하세요"
```

### MessagesPlaceholder — แทรก Chat History

```python
# file: 07_placeholder.py

from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain.messages import HumanMessage, AIMessage

prompt = ChatPromptTemplate.from_messages([
    ("system", "คุณคือผู้ช่วย AI ตอบสั้นๆ"),

    MessagesPlaceholder(variable_name="chat_history"),
    # ▲
    # MessagesPlaceholder = "ช่องว่าง" สำหรับใส่ messages
    # variable_name = ชื่อตัวแปรที่จะใส่ค่าทีหลัง
    # ใช้สำหรับแทรก chat history ที่ไม่รู้จำนวนล่วงหน้า

    ("human", "{question}"),
])

# ใส่ค่า
formatted = prompt.invoke({
    "chat_history": [
        HumanMessage(content="ผมชื่อ สมชาย"),
        AIMessage(content="สวัสดีครับ คุณสมชาย!"),
    ],
    "question": "ผมชื่ออะไร?"
})

response = model.invoke(formatted)
print(response.content)  # "คุณชื่อ สมชาย ครับ"
```

---

## 1.7 Chains (LCEL) — เชื่อมขั้นตอนด้วย |

### LCEL คืออะไร

```python
# LCEL = LangChain Expression Language
# ใช้เครื่องหมาย | (pipe) เชื่อมขั้นตอนเข้าด้วยกัน
# เหมือน Unix pipe: cat file.txt | grep "hello" | wc -l

# ขั้นตอน:  สร้าง prompt → ส่งให้ model → ดึงแค่ text
# เขียนเป็น: prompt | model | parser
```

### ตัวอย่าง Chain

```python
# file: 08_chain.py

from dotenv import load_dotenv
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

load_dotenv()
model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")

# ---- สร้าง Components ----
prompt = ChatPromptTemplate.from_messages([
    ("system", "คุณคือกวีที่เขียนกลอนภาษาไทย"),
    ("human", "เขียนกลอนสั้นๆ เรื่อง {topic}"),
])

parser = StrOutputParser()
# StrOutputParser คืออะไร?
# = ดึงแค่ .content (string) ออกมาจาก AIMessage
# ไม่มี parser: ได้ AIMessage object
# มี parser:    ได้ string ตรงๆ

# ---- สร้าง Chain ด้วย | ----
chain = prompt | model | parser
#       ▲         ▲       ▲
#       │         │       │
#   สร้าง prompt  │   ดึงแค่ text
#   จาก template  │
#              ส่งให้ AI

# ---- ใช้ Chain ----
result = chain.invoke({"topic": "ฝนตก"})
print(result)
# Output: "ฝนโปรยปรายลงมา ท้องฟ้าหม่นหมอง..."
# → ได้ string เลย ไม่ต้อง .content

# ---- Chain ก็มี stream, batch ได้ ----
# stream
for chunk in chain.stream({"topic": "แมว"}):
    print(chunk, end="", flush=True)

# batch
results = chain.batch([
    {"topic": "ฝน"},
    {"topic": "ทะเล"},
    {"topic": "ภูเขา"},
])
for r in results:
    print(r)
```

---

## 1.8 Structured Output — บังคับ AI ตอบเป็น JSON/Object

### ปัญหาของ Free Text

```python
# ❌ ถ้าขอให้ AI ตอบเป็น JSON แบบปกติ
response = model.invoke("สร้างข้อมูลสัตว์เลี้ยง 1 ตัว ตอบเป็น JSON")
print(response.content)
# อาจได้:
# "```json\n{\"name\": \"มิกกี้\"}\n```"   ← มี ``` ปน
# หรือ "ข้อมูลสัตว์เลี้ยง: {name: มิกกี้}" ← format ผิด
# → ไม่แน่นอน! นำไปใช้ในโปรแกรมยาก
```

### with_structured_output — วิธีที่ถูกต้อง

```python
# file: 09_structured.py

from dotenv import load_dotenv
from langchain_google_genai import ChatGoogleGenerativeAI
from pydantic import BaseModel, Field

load_dotenv()
model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")

# ---- 1. กำหนด Schema ด้วย Pydantic ----
class Pet(BaseModel):
    """ข้อมูลสัตว์เลี้ยง"""
    # BaseModel คือ class จาก pydantic ที่ช่วยกำหนดโครงสร้างข้อมูล

    name: str = Field(description="ชื่อสัตว์เลี้ยง")
    # name: str → ต้องเป็น string
    # Field(description="...") → คำอธิบายให้ AI เข้าใจ

    animal_type: str = Field(description="ประเภทสัตว์ เช่น แมว สุนัข กระต่าย")
    age: int = Field(description="อายุ (ปี)", ge=0, le=30)
    #                                          ▲       ▲
    #                                     ge = greater than or equal (>=0)
    #                                     le = less than or equal (<=30)
    #                                     = validation อัตโนมัติ

    color: str = Field(description="สี")
    is_vaccinated: bool = Field(description="ฉีดวัคซีนแล้วหรือยัง")
    favorite_foods: list[str] = Field(description="อาหารที่ชอบ 2-3 อย่าง")

# ---- 2. บอก Model ให้ตอบตาม Schema ----
structured_model = model.with_structured_output(Pet)
#                        ▲
#                   with_structured_output(Schema)
#                   = บังคับให้ AI ตอบตาม Schema ที่กำหนด

# ---- 3. ใช้งาน ----
pet = structured_model.invoke("สร้างข้อมูลแมวสีส้มชื่อ มิกกี้ อายุ 3 ปี")

# pet เป็น Pet object (ไม่ใช่ string!)
print(f"ชื่อ: {pet.name}")                # มิกกี้
print(f"ประเภท: {pet.animal_type}")       # แมว
print(f"อายุ: {pet.age} ปี")              # 3
print(f"สี: {pet.color}")                 # ส้ม
print(f"ฉีดวัคซีน: {pet.is_vaccinated}")  # True
print(f"อาหารโปรด: {pet.favorite_foods}") # ['ปลาทูน่า', 'ไก่', 'ขนมแมว']

# แปลงเป็น dict
print(pet.model_dump())
# {'name': 'มิกกี้', 'animal_type': 'แมว', 'age': 3, ...}

# แปลงเป็น JSON string
print(pet.model_dump_json(indent=2))
# {
#   "name": "มิกกี้",
#   "animal_type": "แมว",
#   ...
# }
```

### Schema ซับซ้อน — Nested + Enum + List

```python
# file: 10_complex_schema.py

from enum import Enum
from pydantic import BaseModel, Field

# ---- Enum: บังคับให้เลือกจากตัวเลือก ----
class Difficulty(str, Enum):
    EASY = "easy"
    MEDIUM = "medium"
    HARD = "hard"

# ---- Nested Model ----
class Ingredient(BaseModel):
    name: str = Field(description="ชื่อวัตถุดิบ")
    amount: str = Field(description="ปริมาณ เช่น '2 ช้อนโต๊ะ' หรือ '200g'")

class Recipe(BaseModel):
    """สูตรอาหาร"""
    dish_name: str = Field(description="ชื่อเมนู")
    difficulty: Difficulty = Field(description="ระดับความยาก")
    #           ▲
    #      ใช้ Enum → AI ต้องเลือก easy/medium/hard เท่านั้น

    prep_time_minutes: int = Field(description="เวลาเตรียม (นาที)")
    ingredients: list[Ingredient] = Field(description="วัตถุดิบ")
    #            ▲
    #       list ของ Ingredient → ได้ list ของ object ที่มี name + amount

    steps: list[str] = Field(description="ขั้นตอนการทำ")

structured_model = model.with_structured_output(Recipe)
recipe = structured_model.invoke("สูตรข้าวผัดกระเพราหมูสับแบบง่าย")

print(f"🍳 {recipe.dish_name}")
print(f"⏱️  {recipe.prep_time_minutes} นาที | ระดับ: {recipe.difficulty.value}")
print(f"\n📝 วัตถุดิบ:")
for ing in recipe.ingredients:
    print(f"   - {ing.name}: {ing.amount}")
print(f"\n👨‍🍳 ขั้นตอน:")
for i, step in enumerate(recipe.steps, 1):
    print(f"   {i}. {step}")
```

**Output:**
```
🍳 ข้าวผัดกระเพราหมูสับ
⏱️  15 นาที | ระดับ: easy

📝 วัตถุดิบ:
   - หมูสับ: 200g
   - กระเทียม: 5 กลีบ
   - พริกขี้หนู: 5 เม็ด
   - ใบกระเพรา: 1 กำมือ
   - น้ำมันพืช: 2 ช้อนโต๊ะ
   - ซอสหอยนางรม: 1 ช้อนโต๊ะ
   - น้ำปลา: 1 ช้อนโต๊ะ
   - น้ำตาล: 1 ช้อนชา

👨‍🍳 ขั้นตอน:
   1. ตั้งกระทะใส่น้ำมัน รอให้ร้อน
   2. ใส่กระเทียมและพริกลงผัดจนหอม
   3. ใส่หมูสับลงผัดจนสุก
   4. ปรุงรสด้วยซอสหอยนางรม น้ำปลา น้ำตาล
   5. ใส่ใบกระเพราลงผัดพอเข้ากัน ปิดไฟ
```

---

## 1.9 ดู Token Usage — เช็คค่าใช้จ่าย

```python
# file: 11_tokens.py

response = model.invoke("อธิบาย AI สั้นๆ")

# ดู token usage
usage = response.usage_metadata
print(f"Input tokens:  {usage['input_tokens']}")    # จำนวน token ที่ส่งไป
print(f"Output tokens: {usage['output_tokens']}")   # จำนวน token ที่ AI สร้าง
print(f"Total tokens:  {usage['total_tokens']}")     # รวม

# ประมาณค่าใช้จ่าย (Gemini Flash = $0.075 / 1M input tokens)
cost = (usage['input_tokens'] * 0.075 + usage['output_tokens'] * 0.30) / 1_000_000
print(f"Estimated cost: ${cost:.6f}")
```

---

## 1.10 Error Handling — จัดการ Error

```python
# file: 12_errors.py

# ---- with_retry: ลองใหม่อัตโนมัติ ----
model_retry = model.with_retry(
    stop_after_attempt=3,     # ลองสูงสุด 3 ครั้ง
)
response = model_retry.invoke("สวัสดี")
# ถ้า API error ครั้งแรก → จะลองอีก 2 ครั้ง

# ---- with_fallbacks: ใช้ model สำรอง ----
from langchain_google_genai import ChatGoogleGenerativeAI

primary = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")
backup = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")  # หรือเปลี่ยนเป็น OpenAI

safe_model = primary.with_fallbacks([backup])
#                      ▲
#               ถ้า primary ล้มเหลว → ใช้ backup แทน

response = safe_model.invoke("สวัสดี")

# ---- try/except: จัดการ error เอง ----
try:
    response = model.invoke("สวัสดี")
    print(response.content)
except Exception as e:
    print(f"Error: {e}")
    # อาจจะ: log error, ส่ง alert, return default response
```

---

## สรุป Step 1 — สิ่งที่ทำได้แล้ว

```
✅ ติดตั้ง LangChain + เชื่อมต่อ Gemini
✅ คุยกับ AI ได้ (invoke, stream, batch)
✅ ปรับพฤติกรรม AI ด้วย temperature, max_tokens
✅ ใช้ Messages กำหนดบทบาท + สร้างประวัติบทสนทนา
✅ ส่งรูปภาพให้ AI วิเคราะห์
✅ สร้าง Prompt Template ที่ใส่ตัวแปรได้
✅ เชื่อม Chain ด้วย LCEL (prompt | model | parser)
✅ บังคับ AI ตอบเป็น JSON/Object ด้วย Structured Output
✅ ดู token usage
✅ จัดการ error (retry, fallback)
```

### ไฟล์ทั้งหมดที่สร้าง

```
my-ai-project/
├── .env
├── 01_hello.py
├── 02_parameters.py
├── 03_invoke_methods.py
├── 04_messages.py
├── 05_image.py
├── 06_templates.py
├── 07_placeholder.py
├── 08_chain.py
├── 09_structured.py
├── 10_complex_schema.py
├── 11_tokens.py
└── 12_errors.py
```

### ไป Step 2 ต่อ → Tools & Agents
