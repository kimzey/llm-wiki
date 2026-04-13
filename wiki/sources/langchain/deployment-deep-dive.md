---
title: "Deployment Deep Dive — Deploy Agent ขึ้น Cloud"
type: source
source_file: raw/notes/LangChain/deployment-deep-dive.md
tags: [langchain, deployment, langserve, docker, cloud, production]
related: []
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/langchain/deployment-deep-dive.md|Original file]]

## สรุป

เอกสาร Deep Dive สำหรับการ Deploy Agent ขึ้น Cloud ครอบคลุมวิธีการ Deploy ด้วย LangServe & LangGraph Cloud, FastAPI, Docker, และ Cloud Providers ต่างๆ พร้อม Authentication, Scaling, Monitoring, และ Production Checklist

## ประเด็นสำคัญ

### ตัวเลือกในการ Deploy

```
┌────────────────────────────────────────────────────────────┐
│                    Deployment Options                       │
├────────────────────┬───────────────────┬───────────────────┤
│   LangServe        │  LangGraph Cloud  │  Self-hosted      │
│   (REST API)       │  (Managed)        │  (FastAPI/Docker) │
├────────────────────┼───────────────────┼───────────────────┤
│ ✅ ง่ายมาก         │ ✅ ง่าย           │ ⚠️ ปานกลาง       │
│ ✅ Auto docs       │ ✅ Managed infra  │ ✅ ควบคุม 100%    │
│ ✅ Playground      │ ✅ Built-in state │ ✅ ไม่มีค่า vendor│
│ ⚠️ Limited scale   │ ✅ Auto-scaling   │ ⚠️ ดูแลเอง       │
│ ❌ ไม่มี state mgmt│ ✅ Monitoring     │ ⚠️ Scaling manual │
│                    │ 💰 มีค่าบริการ    │                   │
├────────────────────┼───────────────────┼───────────────────┤
│ เหมาะกับ:          │ เหมาะกับ:         │ เหมาะกับ:         │
│ Prototype, MVP     │ Production       │ Enterprise,       │
│ Simple chains      │ Complex agents   │ Custom infra      │
└────────────────────┴───────────────────┴───────────────────┘
```

### LangServe

**LangServe คืออะไร**
- แปลง LangChain chain/agent เป็น **REST API** อัตโนมัติ
- พร้อม Swagger docs และ Playground

```bash
pip install langserve[all] uvicorn
```

**ได้อัตโนมัติ**
- `/docs` → Swagger UI (API docs)
- `/playground` → ลอง API ได้เลย
- `/invoke` → เรียกแบบปกติ
- `/batch` → ส่งหลาย requests พร้อมกัน
- `/stream` → Streaming response

### LangGraph Platform

**คืออะไร**
- Deploy LangGraph บน Cloud ของ LangChain
- Managed infrastructure ไม่ต้องดูแลเอง

**ข้อดี**
- ✅ Managed infra
- ✅ Built-in state management
- ✅ Auto-scaling
- ✅ Monitoring ในตัว
- 💰 มีค่าบริการ

### FastAPI เอง (Self-hosted)

**ข้อดี**
- ✅ ควบคุม 100%
- ✅ ไม่มีค่า vendor lock-in
- ✅ ปรับแต่งได้ตามใจ

**ข้อเสีย**
- ⚠️ ดูแลเอง
- ⚠️ Scaling manual
- ⚠️ ต้อง setup monitoring เอง

### Docker

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "server:app", "--host", "0.0.0.0", "--port", "8000"]
```

### หัวข้อที่ครอบคลุม

1. **Overview** — ตัวเลือกในการ Deploy
2. **LangServe** — สร้าง REST API จาก Chain/Agent
3. **LangGraph Platform** — Deploy Graph บน Cloud
4. **Deploy ด้วย FastAPI เอง**
5. **Deploy ด้วย Docker**
6. **Deploy บน Cloud Providers**
7. **Authentication & Security**
8. **Scaling & Performance**
9. **Monitoring ใน Production**
10. **CI/CD Pipeline**
11. **Cost Optimization**
12. **Production Checklist**

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- **LangServe** = ง่ายที่สุด แค่ 10 บรรทัด → ได้ REST API + Swagger docs + Playground
- **LangGraph Platform** = เหมาะกับ Production จริงจัง ที่ต้องการ Managed infra
- **Self-hosted** = เหมาะกับ Enterprise ที่ต้องการ Custom infra และควบคุมเอง

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อมูลที่ขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/ai-agent|AI Agent]]
- [[wiki/concepts/agentic-rag|Agentic RAG]]
