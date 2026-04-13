---
title: "LangSmith Deep Dive — Observability & Evaluation Platform"
type: source
source_file: raw/notes/LangChain/langsmith-deep-dive.md
tags: [langsmith, observability, evaluation, tracing, monitoring]
related: []
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/langchain/langsmith-deep-dive.md|Original file]]

## สรุป

เอกสาร Deep Dive สำหรับ LangSmith ซึ่งเป็น Platform สำหรับ Tracing, Debugging, Testing, และ Monitoring LLM Applications ช่วยให้รู้ว่า Agent ทำอะไร ทำไมถึงตอบแบบนั้น และจะปรับปรุงอย่างไร

## ประเด็นสำคัญ

### LangSmith คืออะไร

**ปัญหาที่ LangSmith แก้**
```
❌ ปัญหาทั่วไปเวลาสร้าง LLM App:
  - Agent ตอบผิด แต่ไม่รู้ว่าผิดตรงไหน
  - เปลี่ยน prompt แล้ว ไม่รู้ว่าดีขึ้นหรือแย่ลง
  - ใน production ไม่รู้ว่า user ถามอะไร AI ตอบอะไร
  - เรียก API ช้า แต่ไม่รู้ว่าช้าที่ขั้นตอนไหน
  - ไม่รู้ว่าใช้ token ไปเท่าไหร่

✅ LangSmith แก้ปัญหาเหล่านี้ด้วย:
  - Tracing: ดู trace ทุก step แบบ real-time
  - Evaluation: วัดคุณภาพด้วย dataset + evaluator
  - Monitoring: dashboard สำหรับ production
  - Prompt Management: จัดการ prompt version
```

### ความสามารถของ LangSmith

| ความสามารถ | รายละเอียด |
|-----------|----------|
| **Tracing** | ดูทุก LLM call, tool call, chain step พร้อม input/output/latency/tokens |
| **Debugging** | คลิกเข้าไปดูแต่ละ step ว่าส่ง prompt อะไร ได้คำตอบอะไร |
| **Evaluation** | สร้างชุดทดสอบ รัน eval วัดคุณภาพแบบอัตโนมัติ |
| **Comparison** | เปรียบเทียบ 2 versions ว่าอันไหนดีกว่า |
| **Monitoring** | ดู latency, error rate, token usage แบบ real-time |
| **Annotation** | ให้คนมา label/review ผลลัพธ์ของ AI |
| **Prompt Hub** | เก็บ prompt แบบมี version control |

### Setup & Configuration

**ติดตั้ง**
```bash
pip install -U langsmith
```

**ตั้งค่า Environment Variables**
```env
LANGSMITH_API_KEY=lsv2_pt_xxxxxxxxxxxxxxxx
LANGSMITH_TRACING=true
LANGSMITH_PROJECT=my-first-project
LANGSMITH_ENDPOINT=https://api.smith.langchain.com
```

**ใน Python**
```python
import os
from dotenv import load_dotenv
load_dotenv()
# แค่นี้! LangChain จะส่ง trace ไป LangSmith อัตโนมัติ
```

### หัวข้อที่ครอบคลุม

1. **LangSmith คืออะไร ทำไมต้องใช้**
2. **Setup & Configuration**
3. **Tracing** — ติดตามทุกขั้นตอน
4. **Projects & Organization**
5. **Datasets** — ชุดข้อมูลทดสอบ
6. **Evaluation** — วัดคุณภาพอย่างเป็นระบบ
7. **Custom Evaluators**
8. **LLM-as-Judge** — ใช้ AI ตรวจ AI
9. **Online Evaluation** — ตรวจใน Production
10. **Monitoring & Alerting**
11. **Prompt Hub** — จัดการ Prompts
12. **Best Practices**

### สมัครและสร้าง API Key

1. ไปที่ [smith.langchain.com](https://smith.langchain.com)
2. สมัครสมาชิก (มี free tier)
3. ไปที่ Settings → API Keys → Create API Key
4. Copy key

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- **Free tier** มีให้ใช้
- **Tracing** ทำงานอัตโนมัติหลังจากตั้งค่า environment variables
- **Projects** ใช้จัดการ organization ของ traces
- **Datasets** ใช้สำหรับ evaluation
- **LLM-as-Judge** = ใช้ LLM ตรวจคุณภาพของ LLM อื่น
- **Prompt Hub** = version control สำหรับ prompts

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อมูลที่ขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/ai-agent|AI Agent]]
- [[wiki/concepts/agentic-rag|Agentic RAG]]
