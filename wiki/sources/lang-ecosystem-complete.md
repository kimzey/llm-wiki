---
title: "Lang Ecosystem ทั้งหมด — ครบทุกตัว"
type: source
source_file: raw/notes/LangChain/lang-ecosystem-complete.md
tags: [langchain, langgraph, langsmith, ecosystem, overview]
related: []
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/LangChain/lang-ecosystem-complete.md|Original file]]

## สรุป

เอกสารสรุปทุกสิ่งในตระกูล Lang (Lang Ecosystem) ว่ามีอะไรบ้าง ทำอะไรได้ ฟรีไหม ใช้เมื่อไหร่ ครอบคลุมทั้ง Python และ JavaScript ecosystems

## ประเด็นสำคัญ

### ภาพรวม Lang Ecosystem

```
┌─────────────────────────────────────────────────────────────────┐
│                      LANG ECOSYSTEM                             │
│                                                                 │
│  ┌─── สร้าง ──────────────────────────────────────────────┐     │
│  │                                                        │     │
│  │  LangChain          LangGraph         LangChain.js     │     │
│  │  (Framework หลัก)    (Flow ซับซ้อน)    (JavaScript)     │     │
│  │                                                        │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
│  ┌─── Deploy ─────────────────────────────────────────────┐     │
│  │                                                        │     │
│  │  LangServe          LangGraph Platform                 │     │
│  │  (REST API)          (Managed Cloud)                   │     │
│  │                                                        │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
│  ┌─── ตรวจสอบ & ปรับปรุง ─────────────────────────────────┐     │
│  │                                                        │     │
│  │  LangSmith          LangGraph Studio                   │     │
│  │  (Trace/Eval/Monitor) (Debug GUI)                      │     │
│  │                                                        │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
│  ┌─── Community & Integrations ───────────────────────────┐     │
│  │                                                        │     │
│  │  LangChain Community    LangChain Hub                  │     │
│  │  (Third-party plugins)  (Prompt marketplace)           │     │
│  │                                                        │     │
│  └────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

### 1. LangChain — Framework หลัก

| | |
|---|---|
| **คือ** | Open Source Framework สำหรับสร้าง AI Application ด้วย LLM |
| **ภาษา** | Python |
| **ฟรี** | ✅ ฟรี 100% (MIT License) |
| **ติดตั้ง** | `pip install langchain` |

**ทำอะไรได้**
- 📦 Models — เชื่อมต่อ LLM ทุกเจ้า
- 💬 Messages — ระบบสื่อสารกับ LLM
- 📝 Prompt Templates — สร้าง Prompt แบบมี Template
- 🔧 Tools — สร้างเครื่องมือให้ AI ใช้
- 🤖 Agents — สร้าง AI Agent ที่คิดเองได้
- 📊 Structured Output — บังคับ Output เป็น JSON/Object
- 🔗 Chains & LCEL — เชื่อมขั้นตอนด้วย pipe |
- 📄 Document Loaders — โหลดข้อมูลจากทุกแหล่ง

### 2. LangGraph — Flow ซับซ้อน

| | |
|---|---|
| **คือ** | Library สำหรับควบคุม Agent Flow ที่ซับซ้อน |
| **ฟรี** | ✅ ฟรี 100% |
| **ติดตั้ง** | `pip install langgraph` |
| **เหมาะกับ** | Multi-Agent, Human-in-the-loop, Complex workflows |

### 3. LangSmith — Observability Platform

| | |
|---|---|
| **คือ** | Platform สำหรับ Tracing, Debugging, Testing, Monitoring |
| **ฟรี** | ✅ มี Free tier |
| **ใช้เมื่อ** | ต้องการ Debug หรือ Monitor AI ใน Production |

### 4. LangServe — REST API

| | |
|---|---|
| **คือ** | Deploy LangChain หรือ LangGraph เป็น REST API |
| **ฟรี** | ✅ ฟรี (Open Source) |
| **ติดตั้ง** | `pip install "langserve[all]"` |
| **ได้อะไร** | REST API + Swagger UI + Playground |

### 5. LangChain.js — JavaScript

| | |
|---|---|
| **คือ** | LangChain version ภาษา JavaScript/TypeScript |
| **ฟรี** | ✅ ฟรี 100% |
| **ติดตั้ง** | `npm install @langchain/core` |
| **เหมาะกับ** | Frontend, Node.js, Next.js, React |

### 6. LangGraph Platform — Managed Cloud

| | |
|---|---|
| **คือ** | Deploy LangGraph บน Cloud ของ LangChain |
| **ราคา** | 💵 มี Free tier และ Paid plans |
| **เหมาะกับ** | Production ที่ต้องการความง่ายในการ Deploy |

### 7. LangGraph Studio — Debug GUI

| | |
|---|---|
| **คือ** | GUI สำหรับ Debug LangGraph |
| **ฟรี** | ✅ ฟรี |
| **ทำอะไร** | Visualize graph, ดู state, trace step-by-step |

### 8. LangChain Community — Third-party Plugins

| | |
|---|---|
| **คือ** | Integrations ที่ community ดูแล |
| **ฟรี** | ✅ ฟรี |
| **ติดตั้ง** | `pip install langchain-community` |

### 9. LangChain Hub — Prompt Marketplace

| | |
|---|---|
| **คือ** | Marketplace สำหรับ Prompts |
| **ฟรี** | ✅ ฟรี |
| **ทำอะไร** | เก็บ prompt แบบมี version control |

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- **ทุกอย่างฟรี** สำหรับ Open Source (LangChain, LangGraph, LangServe, LangChain.js)
- **LangSmith** มี Free tier (ใช้ได้ฟรีจำกัด)
- **LangGraph Platform** มี Free tier + Paid plans
- **LangChain.js** ใช้งานร่วมกับ Python version ได้ (share prompts, datasets)

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อมูลที่ขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/ai-agent|AI Agent]]
- [[wiki/concepts/agentic-rag|Agentic RAG]]
