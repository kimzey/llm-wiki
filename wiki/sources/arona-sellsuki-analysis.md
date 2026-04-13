---
title: "วิเคราะห์ RAG สำหรับ Sellsuki — Internal Company Knowledge Base"
type: source
source_file: raw/notes/arona/SELLSUKI_RAG_ANALYSIS.md
url: ""
published: 2026-04-13
tags: [arona, sellsuki, enterprise-rag, internal-knowledge-base, hr, multi-channel]
related: [wiki/concepts/rag-retrieval-augmented-generation.md, wiki/sources/arona-overview.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/arona/SELLSUKI_RAG_ANALYSIS.md|Original file]]

## สรุป

วิเคราะห์ความเป็นไปได้ในการนำ Arona มาปรับใช้เป็นระบบ RAG สำหรับตอบคำถามภายในบริษัท Sellsuki (HR policies, procedures, business rules) ผ่าน channels ต่างๆ เช่น LINE Bot, Slack, Web Chat

## ประเด็นสำคัญ

### Use Case ที่รองรับ
```
1. Policy Questions:   "ลาพักร้อนกี่วัน?", "โบนัสเมื่อไหร่จ่าย?"
2. How-to Questions:  "ลาหยุดยังไง?", "เคลมประกันยังไง?"
3. Business Rules:    "วิธีคำนวณ commission?"
4. Quick Info:        "WiFi password?", "เบอร์ IT support?"
```

### ความแตกต่างจาก Arona ดั้งเดิม

| ด้าน | Arona (Elysia docs) | Sellsuki RAG |
|------|---------------------|--------------|
| Security | PoW + Turnstile | Auth (LINE/Slack/Employee ID) |
| Access Control | Public | Role-based (HR/Admin/Dev/Employee) |
| Channels | Web only | LINE Bot + Slack + Web + Cursor |
| Documents | Technical docs | HR Policy, Procedures, Business Rules |
| Language | English/Thai | Thai เป็นหลัก |

### Channels ที่ต้องรองรับ
- LINE Bot (พนักงานถาม HR)
- Slack Bot (internal communication)
- Web Chat (HR portal)
- Cursor/IDE (สำหรับ Dev team ถาม technical docs)

### Role-based Access Control
```
Employee:  เห็นเฉพาะ HR policies ทั่วไป
Dev:       เห็น technical docs + HR policies
HR/Admin:  เห็นทั้งหมด รวมข้อมูลเงินเดือน/sensitive docs
```

### ปัญหาก่อนและหลังใช้ RAG
| ประเด็น | ก่อน | หลัง |
|---------|------|------|
| ความเร็ว | รอ HR ช้า | ได้ทันที 24/7 |
| ความถูกต้อง | ตอบผิดหรือจำไม่ได้ | อ้างอิงจากเอกสารจริง |
| Source of Truth | กระจัดกระจาย | เอกสารเดียว อัปเดตได้ |
| Workload HR | ตอบคำถามซ้ำๆ | AI ตอบแทน |

### สรุปความเป็นไปได้
ระบบ Arona สามารถนำมาปรับใช้ได้ 100% โดยเพิ่ม:
1. Authentication layer (แทน PoW + Turnstile)
2. Role-based metadata filtering ใน database
3. Multi-channel adapters (LINE/Slack webhooks)
4. Thai language support (ระบบรองรับอยู่แล้ว)

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG (Retrieval-Augmented Generation)]]
- [[wiki/sources/arona-overview|Arona System Overview]]
