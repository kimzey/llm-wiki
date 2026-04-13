---
title: "Step 1: LangChain พื้นฐาน"
type: source
source_file: raw/notes/LangChain/step1-langchain-basics.md
tags: [langchain, llm, tutorial, basics]
related: []
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/LangChain/step1-langchain-basics.md|Original file]]

## สรุป

บทช่วยสอน LangChain พื้นฐานตั้งแต่ศูนย์ ครอบคลุมการติดตั้ง การใช้งาน LLM ผ่าน LangChain ระบบ Messages และ Prompt Templates เหมาะสำหรับผู้เริ่มต้นที่ยังไม่มีพื้นฐาน AI เนื้อหาครอบคลุมตั้งแต่การเตรียม environment, การเรียก AI ครั้งแรก ไปจนถึงการส่งรูปภาพและการใช้ Prompt Templates

## ประเด็นสำคัญ

### การติดตั้งและเตรียมความพร้อม
- ติดตั้ง Python 3.11+ และสร้าง virtual environment
- ติดตั้ง LangChain: `pip install "langchain[google-genai]" python-dotenv`
- สร้างไฟล์ `.env` เพื่อเก็บ API Key (ห้าม push ขึ้น GitHub)
- ใช้ Google AI Studio สร้าง API Key สำหรับ Gemini

### Model และการเรียกใช้งาน
- `ChatGoogleGenerativeAI` class ใช้เชื่อมต่อกับ Gemini models
- รองรับ model: `gemini-2.5-flash-lite`, `gemini-2.5-pro`
- มี 4 วิธีเรียก: `invoke()`, `stream()`, `batch()`, `ainvoke()`
- Parameters สำคัญ: `temperature`, `max_output_tokens`, `top_p`, `timeout`, `max_retries`

### Messages และ Roles
- **SystemMessage**: กำหนดบทบาท/กฎให้ AI
- **HumanMessage**: ข้อความจากผู้ใช้
- **AIMessage**: คำตอบจาก AI (ใช้เมื่อจำลองประวัติ)
- **ToolMessage**: ผลลัพธ์จาก Tool
- สามารถเขียนเป็น dictionary format: `{"role": "system", "content": "..."}`

### Temperature
- **0.0**: ตอบเหมือนเดิมทุกครั้ง (เหมาะกับตอบคำถาม, คำนวณ)
- **0.5**: balance
- **1.0**: สร้างสรรค์ สุ่มมาก (เหมาะกับเขียนเรื่อง, brainstorm)

### Chat History
- AI ไม่มีความจำเอง ต้องส่งประวัติทุกครั้ง
- ใส่ทุก message ที่เคยมี (system + user + assistant + user...)
- ปกติเก็บแค่ 20 messages ล่าสุด เพื่อประหยัด token

### Prompt Templates
- ใช้ `ChatPromptTemplate.from_messages()` สร้าง template
- รองรับตัวแปร: `{language}`, `{text}` ฯลฯ
- ใส่ค่าตัวแปรด้วย `.invoke({"language": "English", "text": "..."})`
- ช่วยลด code ซ้ำ

### การส่งรูปภาพ (Multimodal)
- ส่งรูปจากไฟล์: แปลงเป็น base64
- ส่งรูปจาก URL: ใช้ `{"type": "image_url", "image_url": {"url": "..."}}`
- Content สามารถเป็น list: `[{"type": "text", "text": "..."}, {"type": "image", ...}]`

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- 1 token ≈ 1 คำภาษาอังกฤษ / 1-2 ตัวอักษรภาษาไทย
- `temperature=0` ถามซ้ำ 3 รอบ → ได้คำตอบเหมือนกัน
- `temperature=1` ถามซ้ำ 3 รอบ → ได้คำตอบต่างกัน
- `stream()` method ใช้ทำ chatbot ให้ user เห็นคำตอบทีละคำ (UX ดีกว่า)
- `batch()` ประหยัดเวลากว่า invoke ทีละตัว
- Dictionary format นิยมกว่า class format เพราะอ่านง่าย แยกเป็น JSON ได้

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อมูลที่ขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/llm-large-language-model|LLM (Large Language Model)]]
- [[wiki/concepts/ai-agent|AI Agent]]
- [[wiki/concepts/agentic-rag|Agentic RAG]]
