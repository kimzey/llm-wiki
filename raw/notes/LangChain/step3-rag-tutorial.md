# Step 3: RAG — ให้ AI ตอบจากเอกสารของเรา
## Retrieval-Augmented Generation ตั้งแต่ศูนย์

> Step นี้จะทำให้ AI ตอบจาก "เอกสารของเรา" ได้ เช่น คู่มือบริษัท, FAQ, นโยบาย

---

## 3.1 RAG คืออะไร (อธิบายง่ายๆ)

```
ปัญหา:
  AI รู้แค่ข้อมูลที่มันถูกฝึกมา (knowledge cutoff)
  AI ไม่รู้ข้อมูลภายในบริษัทเรา

RAG แก้ยังไง:
  1. เอาเอกสารของเรา → ตัดเป็นชิ้นเล็กๆ → เก็บในฐานข้อมูลพิเศษ
  2. พอมีคำถาม → ค้นหาชิ้นที่เกี่ยวข้อง
  3. ส่งชิ้นที่เกี่ยวข้อง + คำถาม ให้ AI → AI ตอบจากข้อมูลจริง

เปรียบเทียบ:
  AI ปกติ  = นักเรียนทำข้อสอบจากความจำ (อาจจำผิด)
  AI + RAG = นักเรียนทำข้อสอบแบบเปิดหนังสือ (ตอบจากข้อมูลจริง)
```

### Pipeline

```
เอกสาร → [ตัดเป็นชิ้น] → [แปลงเป็นตัวเลข] → [เก็บในฐานข้อมูล]
                                                      │
คำถาม → [แปลงเป็นตัวเลข] → [ค้นหาชิ้นที่คล้าย] ─────┘
                                    │
                              ชิ้นที่เกี่ยวข้อง
                                    │
                           [ส่งให้ AI + คำถาม]
                                    │
                               คำตอบสุดท้าย
```

---

## 3.2 ติดตั้ง

```bash
pip install langchain-chroma langchain-text-splitters
# langchain-chroma     = Vector Store (ฐานข้อมูลสำหรับเก็บ embeddings)
# langchain-text-splitters = ตัดเอกสารเป็นชิ้น
```

---

## 3.3 ทำ RAG ทีละขั้นตอน

### ขั้นตอนที่ 1: เตรียมเอกสาร

```python
# file: 19_rag.py

from langchain_core.documents import Document

# Document คืออะไร?
# = object ที่เก็บเนื้อหา + metadata ของเอกสาร

documents = [
    Document(
        page_content="""นโยบายการลาของบริษัท XYZ (ปี 2025)
ลาพักร้อน: พนักงานทุกคนมีสิทธิ์ลาพักร้อน 15 วันทำการต่อปี
สะสมวันลาได้ไม่เกิน 5 วัน ต่อไปยังปีถัดไป
แจ้งล่วงหน้าอย่างน้อย 3 วันทำการ
ลาป่วย: 30 วัน/ปี ลาเกิน 3 วันต้องมีใบรับรองแพทย์
ลากิจ: 5 วัน/ปี แจ้งล่วงหน้า 1 วัน""",
        #  ▲
        # page_content = เนื้อหาของเอกสาร

        metadata={"source": "hr-policy.pdf", "category": "leave"}
        #  ▲
        # metadata = ข้อมูลเพิ่มเติม (ไว้อ้างอิงทีหลัง)
    ),

    Document(
        page_content="""สวัสดิการพนักงาน
ประกันสุขภาพกลุ่ม: ครอบคลุมพนักงานและครอบครัว
OPD วงเงิน 3,000 บาท/ครั้ง
IPD วงเงิน 200,000 บาท/ครั้ง
ค่าอาหาร: 100 บาท/วันทำการ
ค่าเดินทาง: 2,500 บาท/เดือน
โบนัส: 0-4 เดือน ตามผลงาน""",
        metadata={"source": "benefits.pdf", "category": "benefits"}
    ),

    Document(
        page_content="""คู่มือ IT
รหัสผ่านต้องมีอย่างน้อย 12 ตัวอักษร
เปลี่ยนรหัสผ่านทุก 90 วัน
ลืมรหัสผ่าน: ไปที่ https://portal.xyz.com/reset
VPN: ดาวน์โหลด FortiClient, server: vpn.xyz.com
แจ้งปัญหา IT: ext. 1234 หรือ helpdesk.xyz.com""",
        metadata={"source": "it-manual.pdf", "category": "it"}
    ),
]

print(f"📄 เตรียมเอกสาร {len(documents)} ฉบับ")
```

### ขั้นตอนที่ 2: ตัดเอกสารเป็น Chunks

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

# ทำไมต้องตัด?
# - Embedding model ทำงานดีกับ text สั้น
# - ค้นหาได้แม่นยำกว่า (เจอส่วนที่เกี่ยวข้องจริงๆ)
# - ประหยัด token (ส่งแค่ส่วนที่เกี่ยวข้อง ไม่ต้องส่งทั้งเอกสาร)

splitter = RecursiveCharacterTextSplitter(
    chunk_size=200,
    # chunk_size คืออะไร?
    # = ขนาดสูงสุดของแต่ละชิ้น (ตัวอักษร)
    # 200-500 = เหมาะกับ Q&A สั้นๆ
    # 500-1000 = เหมาะกับเอกสารทั่วไป

    chunk_overlap=40,
    # chunk_overlap คืออะไร?
    # = จำนวนตัวอักษรที่ซ้อนทับกันระหว่างชิ้น
    # ทำไมต้องซ้อน? → เพื่อไม่ให้ข้อมูลที่อยู่ "ตรงรอยตัด" หายไป
    #
    # ตัวอย่าง (chunk_size=10, overlap=3):
    # เอกสาร: "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    # chunk 1: "ABCDEFGHIJ"
    # chunk 2: "HIJKLMNOPQ"  ← H,I,J ซ้อนกัน
    # chunk 3: "OPQRSTUVWX"  ← O,P,Q ซ้อนกัน

    separators=["\n\n", "\n", ".", " ", ""],
    # separators คืออะไร?
    # = ลำดับที่จะพยายามตัด (พยายามตัดที่ใหญ่ก่อน)
    # "\n\n" = ย่อหน้า (ดีที่สุด ตัดตรงจุดแบ่งเนื้อหา)
    # "\n"   = บรรทัดใหม่
    # "."    = จุด (จบประโยค)
    # " "    = เว้นวรรค
    # ""     = ตัดทีละตัวอักษร (ทางเลือกสุดท้าย)
)

chunks = splitter.split_documents(documents)
#        ▲
#   split_documents() = ตัดแล้วรักษา metadata ไว้ให้

print(f"✂️  ตัดได้ {len(chunks)} chunks")
for i, chunk in enumerate(chunks):
    print(f"\n--- Chunk {i+1} ({len(chunk.page_content)} chars) ---")
    print(f"Source: {chunk.metadata['source']}")
    print(f"Content: {chunk.page_content[:80]}...")
```

**Output:**
```
✂️  ตัดได้ 8 chunks

--- Chunk 1 (185 chars) ---
Source: hr-policy.pdf
Content: นโยบายการลาของบริษัท XYZ (ปี 2025)
ลาพักร้อน: พนักงานทุกคนมีสิทธิ์ลาพ...

--- Chunk 2 (142 chars) ---
Source: hr-policy.pdf
Content: แจ้งล่วงหน้าอย่างน้อย 3 วันทำการ
ลาป่วย: 30 วัน/ปี ลาเกิน 3 วันต้องมี...
...
```

### ขั้นตอนที่ 3: Embedding — แปลงเป็น Vector

```python
from langchain_google_genai import GoogleGenerativeAIEmbeddings

# Embedding คืออะไร?
# = แปลง "ข้อความ" ให้เป็น "ตัวเลข" (Vector)
# เพื่อให้คอมพิวเตอร์เปรียบเทียบ "ความหมาย" ได้
#
# "แมว" → [0.12, -0.45, 0.78, 0.23, ...]   (768 ตัวเลข)
# "สุนัข" → [0.15, -0.42, 0.75, 0.20, ...]  ← ใกล้กัน! (สัตว์เลี้ยงเหมือนกัน)
# "รถยนต์" → [0.85, 0.33, -0.21, 0.67, ...]  ← ไกล! (คนละเรื่อง)

embeddings = GoogleGenerativeAIEmbeddings(model="models/text-embedding-004")
#            ▲
#       Google Embedding model (ฟรี!)

# ทดลอง embed ข้อความ
vector = embeddings.embed_query("แมว")
print(f"Dimensions: {len(vector)}")     # 768
print(f"ตัวอย่าง: {vector[:5]}")         # [0.012, -0.045, 0.078, ...]
```

### ขั้นตอนที่ 4: Vector Store — จัดเก็บ

```python
from langchain_chroma import Chroma

# Vector Store คืออะไร?
# = ฐานข้อมูลพิเศษที่เก็บ embeddings
# ทำให้ค้นหา "ข้อความที่มีความหมายคล้ายกัน" ได้เร็วมาก

vectorstore = Chroma.from_documents(
    documents=chunks,          # chunks ที่ตัดแล้ว
    embedding=embeddings,      # embedding model
    collection_name="company-docs",
    persist_directory="./chroma_db",  # บันทึกลงดิสก์
    #  ▲
    # persist_directory = เก็บลงไฟล์ ไม่ต้องสร้างใหม่ทุกครั้ง
)

print(f"✅ สร้าง Vector Store เรียบร้อย ({len(chunks)} vectors)")
```

### ขั้นตอนที่ 5: Retrieval — ค้นหา

```python
# สร้าง retriever
retriever = vectorstore.as_retriever(
    search_type="similarity",
    # search_type คืออะไร?
    # "similarity" = ค้นหา chunks ที่ vector ใกล้กับคำถามมากที่สุด
    # "mmr"        = ค้นหาแบบมีความหลากหลาย (ไม่เอาอันซ้ำกัน)

    search_kwargs={"k": 3},
    # k คืออะไร?
    # = จำนวน chunks ที่จะดึงมา
    # k=3 = ดึง 3 chunks ที่เกี่ยวข้องที่สุด
)

# ทดสอบค้นหา
results = retriever.invoke("ลาพักร้อนได้กี่วัน")
#         ▲
#    invoke(คำถาม) → return list ของ Document ที่เกี่ยวข้อง

print(f"🔍 ค้นหา: 'ลาพักร้อนได้กี่วัน'")
print(f"📊 พบ {len(results)} chunks:\n")
for i, doc in enumerate(results, 1):
    print(f"  [{i}] ({doc.metadata['source']})")
    print(f"      {doc.page_content[:100]}...")
    print()
```

**Output:**
```
🔍 ค้นหา: 'ลาพักร้อนได้กี่วัน'
📊 พบ 3 chunks:

  [1] (hr-policy.pdf)
      นโยบายการลาของบริษัท XYZ (ปี 2025)
      ลาพักร้อน: พนักงานทุกคนมีสิทธิ์ลาพักร้อน 15 วัน...

  [2] (hr-policy.pdf)
      แจ้งล่วงหน้าอย่างน้อย 3 วันทำการ
      ลาป่วย: 30 วัน/ปี...

  [3] (benefits.pdf)
      สวัสดิการพนักงาน ประกันสุขภาพกลุ่ม...
```

### ขั้นตอนที่ 6: Generation — สร้างคำตอบ

```python
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain_core.prompts import ChatPromptTemplate

model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

# ---- RAG Prompt ----
rag_prompt = ChatPromptTemplate.from_messages([
    ("system", """คุณคือผู้ช่วย AI ของบริษัท XYZ ตอบจากข้อมูลที่ให้มาเท่านั้น

กฎ:
1. ตอบจากข้อมูลอ้างอิงเท่านั้น ห้ามแต่งเพิ่ม
2. ถ้าไม่มีข้อมูล → บอกว่า "ไม่มีข้อมูลในเอกสาร"
3. อ้างอิงเอกสารที่ใช้ตอบ
4. ตอบเป็นภาษาไทย สุภาพ

ข้อมูลอ้างอิง:
{context}"""),

    ("human", "{question}"),
])

# ---- RAG Function ----
def ask(question: str) -> str:
    """ถาม-ตอบแบบ RAG"""

    # Step 1: ค้นหา chunks ที่เกี่ยวข้อง
    docs = retriever.invoke(question)

    # Step 2: รวม chunks เป็น context
    context = "\n\n---\n\n".join([
        f"[{doc.metadata['source']}] {doc.page_content}"
        for doc in docs
    ])

    # Step 3: สร้าง prompt
    formatted = rag_prompt.invoke({
        "context": context,
        "question": question,
    })

    # Step 4: ส่งให้ AI
    response = model.invoke(formatted)

    return response.content

# ---- ทดสอบ ----
questions = [
    "ลาพักร้อนได้กี่วัน?",
    "ลืมรหัสผ่านต้องทำยังไง?",
    "OPD วงเงินเท่าไหร่?",
    "วิธีทำส้มตำ?",           # ← นอกขอบเขต
]

for q in questions:
    print(f"❓ {q}")
    print(f"💬 {ask(q)}")
    print()
```

**Output:**
```
❓ ลาพักร้อนได้กี่วัน?
💬 พนักงานมีสิทธิ์ลาพักร้อน 15 วันทำการต่อปีค่ะ
   สามารถสะสมวันลาได้ไม่เกิน 5 วัน และต้องแจ้งล่วงหน้า
   อย่างน้อย 3 วันทำการ (อ้างอิง: hr-policy.pdf)

❓ ลืมรหัสผ่านต้องทำยังไง?
💬 ให้ไปที่ https://portal.xyz.com/reset ค่ะ
   หรือโทรแจ้ง IT Help Desk ext. 1234 (อ้างอิง: it-manual.pdf)

❓ OPD วงเงินเท่าไหร่?
💬 OPD วงเงิน 3,000 บาท/ครั้งค่ะ (อ้างอิง: benefits.pdf)

❓ วิธีทำส้มตำ?
💬 ขออภัยค่ะ ไม่มีข้อมูลเรื่องวิธีทำส้มตำในเอกสารของบริษัท
```

---

## 3.4 RAG ด้วย LCEL Chain (วิธีสั้น)

```python
# file: 20_rag_chain.py

from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

def format_docs(docs):
    """แปลง list ของ docs → string"""
    return "\n\n".join([d.page_content for d in docs])

# สร้าง RAG Chain ใน 1 บรรทัด!
rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    #           ▲           ▲                         ▲
    #      ค้นหา docs    แปลงเป็น text         ส่ง question ต่อไปตรงๆ
    | rag_prompt
    | model
    | StrOutputParser()
)

# ใช้งาน — สั้นมาก!
answer = rag_chain.invoke("โบนัสได้กี่เดือน?")
print(answer)
```

---

## 3.5 โหลดเอกสารจากไฟล์จริง

```python
# file: 21_load_files.py

# ---- โหลด PDF ----
# pip install pypdf
from langchain_community.document_loaders import PyPDFLoader

loader = PyPDFLoader("docs/handbook.pdf")
docs = loader.load()
# แต่ละหน้า = 1 Document
# metadata จะมี: {'source': 'docs/handbook.pdf', 'page': 0}

# ---- โหลด Text ----
from langchain_community.document_loaders import TextLoader

loader = TextLoader("docs/faq.txt", encoding="utf-8")
docs = loader.load()

# ---- โหลดทุกไฟล์ใน folder ----
from langchain_community.document_loaders import DirectoryLoader

loader = DirectoryLoader("docs/", glob="**/*.pdf", show_progress=True)
docs = loader.load()

# จากนั้นทำเหมือนเดิม: split → embed → store → retrieve → generate
chunks = splitter.split_documents(docs)
vectorstore = Chroma.from_documents(chunks, embeddings)
retriever = vectorstore.as_retriever()
```

---

## 3.6 โหลด Vector Store ที่มีอยู่แล้ว (ไม่ต้องสร้างใหม่)

```python
# ครั้งแรก: สร้าง vector store
vectorstore = Chroma.from_documents(chunks, embeddings, persist_directory="./chroma_db")

# ครั้งถัดไป: โหลดจากดิสก์ (เร็วมาก ไม่ต้อง embed ใหม่!)
vectorstore = Chroma(
    collection_name="company-docs",
    embedding_function=embeddings,
    persist_directory="./chroma_db",
)
retriever = vectorstore.as_retriever()
```

---

## สรุป Step 3 — สิ่งที่ทำได้แล้ว

```
✅ เข้าใจ RAG Pipeline ทั้งหมด
✅ สร้าง Document จาก text / PDF / folder
✅ ตัดเอกสารเป็น chunks (RecursiveCharacterTextSplitter)
✅ เข้าใจ Embedding คืออะไร ทำไมต้องใช้
✅ สร้าง Vector Store ด้วย Chroma
✅ ค้นหา chunks ที่เกี่ยวข้องด้วย Retriever
✅ สร้าง RAG Prompt ที่บอก AI ให้ตอบจาก context เท่านั้น
✅ สร้าง RAG Chain ด้วย LCEL
✅ โหลด Vector Store ที่มีอยู่แล้ว (ไม่ต้อง embed ซ้ำ)
✅ AI ตอบจากเอกสารได้ + ปฏิเสธเมื่อไม่มีข้อมูล
```

### ไป Step 4 ต่อ → LangGraph (ควบคุม Flow ซับซ้อน)
