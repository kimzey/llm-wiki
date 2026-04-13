---
title: "Step 3: RAG — Retrieval-Augmented Generation"
type: source
source_file: raw/notes/LangChain/step3-rag-tutorial.md
tags: [langchain, rag, tutorial, vector-store, retrieval]
related: []
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/LangChain/step3-rag-tutorial.md|Original file]]

## สรุป

บทช่วยสอน RAG (Retrieval-Augmented Generation) ตั้งแต่ศูนย์ สอนวิธีทำให้ AI ตอบจากเอกสารของเราเองได้ เช่น คู่มือบริษัท, FAQ, นโยบาย เนื้อหาครอบคลุม pipeline ทั้งหมด: การโหลดเอกสาร, การตัดเป็น chunks, การแปลงเป็น vectors, การจัดเก็บใน Vector Store, การค้นหา และการสร้างคำตอบ

## ประเด็นสำคัญ

### RAG คืออะไร?
- **ปัญหา**: AI รู้แค่ข้อมูลที่ถูกฝึกมา (knowledge cutoff)
- **วิธีแก้ RAG**:
  1. เอาเอกสาร → ตัดเป็นชิ้นเล็กๆ → เก็บในฐานข้อมูลพิเศษ
  2. พอมีคำถาม → ค้นหาชิ้นที่เกี่ยวข้อง
  3. ส่งชิ้นที่เกี่ยวข้อง + คำถาม ให้ AI → AI ตอบจากข้อมูลจริง
- เปรียบเทียบ: AI ปกติ = ทำข้อสอบจากความจำ, AI + RAG = ทำข้อสอบแบบเปิดหนังสือ

### Pipeline ทั้งหมด
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

### Document
- `page_content`: เนื้อหาของเอกสาร
- `metadata`: ข้อมูลเพิ่มเติม (ไว้อ้างอิงทีหลัง)

### Chunking (RecursiveCharacterTextSplitter)
- ทำไมต้องตัด?
  - Embedding model ทำงานดีกับ text สั้น
  - ค้นหาได้แม่นยำกว่า
  - ประหยัด token
- Parameters:
  - `chunk_size=200`: ขนาดสูงสุดของแต่ละชิ้น (200-500 สำหรับ Q&A, 500-1000 สำหรับเอกสารทั่วไป)
  - `chunk_overlap=40`: จำนวนตัวอักษรที่ซ้อนทับ (เพื่อไม่ให้ข้อมูลที่อยู่ "ตรงรอยตัด" หายไป)
  - `separators=["\n\n", "\n", ".", " ", ""]`: ลำดับที่จะพยายามตัด (พยายามตัดที่ใหญ่ก่อน)

### Embedding
- แปลง "ข้อความ" → "ตัวเลข" (Vector)
- ทำให้คอมพิวเตอร์เปรียบเทียบ "ความหมาย" ได้
- "แมว" → `[0.12, -0.45, 0.78, ...]` (768 ตัวเลข)
- ใช้ `GoogleGenerativeAIEmbeddings(model="models/text-embedding-004")` (ฟรี!)

### Vector Store (Chroma)
- ฐานข้อมูลพิเศษที่เก็บ embeddings
- ทำให้ค้นหา "ข้อความที่มีความหมายคล้ายกัน" ได้เร็วมาก
- ใช้ `Chroma.from_documents(chunks, embeddings, persist_directory="./chroma_db")`
- `persist_directory`: เก็บลงดิสก์ ไม่ต้องสร้างใหม่ทุกครั้ง

### Retriever
- `search_type="similarity"`: ค้นหา chunks ที่ vector ใกล้กับคำถามมากที่สุด
- `search_type="mmr"`: ค้นหาแบบมีความหลากหลาย (ไม่เอาอันซ้ำกัน)
- `search_kwargs={"k": 3}`: จำนวน chunks ที่จะดึงมา

### RAG Prompt
- ต้องบอก AI ให้:
  1. ตอบจากข้อมูลอ้างอิงเท่านั้น ห้ามแต่งเพิ่ม
  2. ถ้าไม่มีข้อมูล → บอกว่า "ไม่มีข้อมูลในเอกสาร"
  3. อ้างอิงเอกสารที่ใช้ตอบ

### LCEL Chain
```python
rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | rag_prompt
    | model
    | StrOutputParser()
)
```

### โหลดเอกสารจากไฟล์จริง
- `PyPDFLoader`: โหลด PDF (แต่ละหน้า = 1 Document)
- `TextLoader`: โหลด text file
- `DirectoryLoader`: โหลดทุกไฟล์ใน folder

### โหลด Vector Store ที่มีอยู่แล้ว
- ครั้งแรก: สร้าง vector store
- ครั้งถัดไป: โหลดจากดิสก์ (เร็วมาก ไม่ต้อง embed ซ้ำ)

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- Chunk overlap ช่วยไม่ให้ข้อมูลที่อยู่ตรงรอยตัดหาย
- 1 token ≈ 1 คำภาษาอังกฤษ / 1-2 ตัวอักษรภาษาไทย
- ค้นหา "ลาพักร้อนได้กี่วัน" → เจอ chunks เรื่องลาพักร้อน + สวัสดิการ
- คำถาม "วิธีทำส้มตำ?" → AI ตอบ "ไม่มีข้อมูลเรื่องนี้ในเอกสารของบริษัท" (ปฏิเสธอย่างถูกต้อง)

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อมูลที่ขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/agentic-rag.md|Agentic RAG]]
