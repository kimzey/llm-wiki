---
title: "DIY — สร้าง RAG ตอบคำถามเกี่ยวกับ Sellsuki"
type: source
source_file: raw/notes/rag-knowledge/[DIY] สร้าง "RAG ที่ตอบคำถามเกี่ยวกับบริษัท Sellsuki ได้" DIY.md
tags: [rag, diy, sellsuki, use-case, implementation]
related: [wiki/concepts/rag-retrieval-augmented-generation]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/rag/[DIY] สร้าง "RAG ที่ตอบคำถามเกี่ยวกับบริษัท Sellsuki ได้" DIY.md|Original file]]

## สรุป

เอกสารนำร่องสร้าง RAG System สำหรับตอบคำถามเกี่ยวกับบริษัท Sellsuki — ครอบคลุม use case, tech stack, และแนวทางการทำ

## ประเด็นสำคัญ

### Use Case

#### ปัจจุบัน (Before)
- พนักงานมีคำถาม → ถาม HR / ถามเพื่อน / เปิดไฟล์หาเอง
- ช้า, ได้คำตอบไม่ตรง, HR เสียเวลาตอบคำถามซ้ำๆ

#### หลังทำ (After)
- พนักงานมีคำถาม → ถาม AI ใน LINE/Slack/Web → ได้คำตอบทันที 24/7
- เร็ว, ตรงประเด็น, อ้างอิงแหล่งที่มาได้, HR ไม่ต้องตอบซ้ำ

### ตัวอย่าง Use Cases

#### ก่อนทำ
- ❌ "ลาพักร้อนกี่วัน?" → ถาม HR → HR ตอบเอง → ช้า
- ❌ "WiFi password อะไร?" → ถามรอบข้าง → ไม่มีใครจำได้
- ❌ "สร้าง order ใน Sellsuki ยังไง?" → เปิดหา docs เอง → หาไม่เจอ
- ❌ Dev: เขียนโค้ด → ไม่รู้ business rule → ถามคนอื่น → รอ

#### หลังทำ
- ✅ "ลาพักร้อนกี่วัน?" → ถาม LINE Bot → "10 วัน/ปี" ทันที
- ✅ "WiFi password อะไร?" → ถาม Slack Bot → ได้ทันที
- ✅ "สร้าง order ยังไง?" → ถาม Web Chat → ได้ขั้นตอนครบ
- ✅ Dev: เขียนโค้ด → ถามใน Cursor → ได้ business rule + ตัวอย่างโค้ด
- ✅ HR: ไม่ต้องตอบคำถามซ้ำๆ → มีเวลาทำงานอื่น

### Tech Stack

#### เทคนิคที่จะทดลอง
- มีการทำ **chunks** หลายๆ แบบ
- พร้อมแนบ **Metadata** ด้วย (สำหรับ filter, search, จำกัดสิทธิ์)
- การทำ **Database & Index** ของ RAG
- จะยัง **ไม่ใช้ Framework** พวก LangChain — จะลองทำด้วยตัวเองก่อนให้เข้าใจ
- มีการทำ **Caching** เช่น **Semantic Cache**
- รองรับการ **update** ข้อมูล (ถ้ามีการ update → อาจจะ Chunk ใหม่ → แต่ถ้า Embedding แล้วมันคือข้อมูลเดิม → ไม่ต้อง Embedding ซ้ำ)

#### Components
- **Vector DB**: ParadeDB
- **Cache**: Dragonfly (Semantic cache, Embedding cache)

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG — Retrieval-Augmented Generation]]
- [[wiki/concepts/vector-database|Vector Database]]
- [[wiki/concepts/semantic-caching|Semantic Caching]]
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search]]
