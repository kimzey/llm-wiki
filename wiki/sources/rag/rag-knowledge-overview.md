---
title: "RAG Agent Knowledge — Overview & Document Statistics"
type: source
source_file: raw/notes/rag-knowledge/00-OVERVIEW.md
tags: [rag, overview, documentation, knowledge-base]
related: [wiki/concepts/rag-retrieval-augmented-generation]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/rag/00-OVERVIEW.md|Original file]]

## สรุป

เอกสารภาพรวมของ RAG Agent Documentation Set ที่รวบรวมเนื้อหาครบถ้วนตั้งแต่พื้นฐานจนถึงการเขียนโปรแกรมจริง — ทั้งหมด 12 ไฟล์ ประมาณ 300 หน้า A4 (16,644 บรรทัด) ครอบคลุมทุกแง่มุมของ RAG & Agent Systems

## ประเด็นสำคัญ

- **ขนาดเอกสาร**: 12 ไฟล์ รวม 16,644 บรรทัด (~300 หน้า A4) เฉลี่ย 1,387 บรรทัด/ไฟล์
- **ระดับความยาก**: แบ่งเป็น Basic, Intermediate, Advanced และ Reference
- **เส้นทางการอ่าน**: 4 path สำหรับผู้เริ่มต้น, ผู้ที่ต้องการ implement, DevOps/SRE, และผู้ที่สนใจ architecture
- **ครอบคลุม**: LLM, RAG, Vector Database, Embedding, Agent, Chunking, Integration, Client Channels, API, Testing, Operations, Security

## รายการเอกสารทั้งหมด

### ระดับพื้นฐาน (Basic)

| ไฟล์ | หน้า | บรรทัด | เนื้อหา |
|------|-----|--------|---------|
| `01-rag-agent-complete-knowledge.md` | ~28 | 1,537 | LLM, RAG, Vector DB, Embedding, Agent — พื้นฐานทั้งหมด |
| `02-rag-glossary-deepdive.md` | ~26 | 1,441 | คำศัพท์ + Data Prep — ศัพท์เฉพาะ + วิธีเตรียม data |

### ระดับกลาง (Intermediate)

| ไฟล์ | หน้า | บรรทัด | เนื้อหา |
|------|-----|--------|---------|
| `03-rag-integration-guide.md` | ~27 | 1,481 | Integration — ต่อกับอะไรได้, Use Case นอก Chat |
| `04-rag-architecture-diagrams.md` | ~29 | 1,597 | Diagram ทั้งหมด — Architecture ทุกมุมมอง |
| `05-sellsuki-agent-guide.md` | ~29 | 1,579 | Implementation Guide — วิธีทำจริงทีละขั้นตอน |
| `06-chunking-deepdive.md` | ~22 | 1,205 | Chunking ลึกฯ — ทุกวิธีตัดข้อความ + เทียบข้อมูลจริง |
| `07-rag-client-channels.md` | ~29 | 1,602 | ช่องทางเชื่อมต่อ — API, Webhook, SDK ทั้งหมด |

### ระดับขั้นสูง (Advanced)

| ไฟล์ | หน้า | บรรทัด | เนื้อหา |
|------|-----|--------|---------|
| `03-sellsuki-agent-plan-v2.md` | ~22 | 1,198 | Plan v2 — Architecture ใหม่และการปรับปรุงจาก v1 |
| `08-rag-vs-agent-explained.md` | ~19 | 1,059 | RAG vs Agent — มันคืออะไร ต่างกันยังไง เชื่อมกันยังไง |

### ระดับอ้างอิง (Reference)

| ไฟล์ | หน้า | บรรทัด | เนื้อหา |
|------|-----|--------|---------|
| `09-api-reference.md` | ~21 | 1,171 | API เต็มๆ — Endpoint, Request/Response ทั้งหมด |
| `10-testing-guide.md` | ~19 | 1,059 | ทดสอบ — RAG Testing, QA, Evaluation Metrics |
| `11-operations-guide.md` | ~22 | 1,226 | ดูแลระบบ — Monitoring, Logging, Troubleshooting |
| `12-security-hardening.md` | ~25 | 1,358 | ความปลอดภัย — Security Best Practices |

## เส้นทางการอ่านแนะนำ

### Path 1: มือใหม่ / อยากเข้าใจพื้นฐานก่อน
```
01 → 02 → 04 → 05
```
- เริ่มจากความรู้พื้นฐาน
- ตามด้วยคำศัพท์และ Data Prep
- ดู Diagram ภาพรวม
- อ่านวิธีทำจริง

### Path 2: อยากเริ่ม Implement ทันที
```
05 → 09 → 08 → 07
```
- Implementation Guide เลย
- API Reference
- ช่องทางเชื่อมต่อ
- Integration กับระบบอื่น

### Path 3: ดูแลระบบ Production
```
11 → 10 → 12 → 06
```
- Operations Guide
- Testing & QA
- Security
- Chunking Optimization

### Path 4: เข้าใจ Architecture ลึกๆ
```
03 → 04 → 06 → 07
```
- Plan v2 (Architecture ใหม่)
- Diagrams
- Chunking Deep Dive
- Integration Patterns

## สรุปเนื้อหาแต่ละไฟล์

### 01 — Complete RAG & Agent Knowledge
- LLM ทำงานยังไง
- RAG คืออะไร
- Vector Database คืออะไร
- Embedding คืออะไร
- Agent คืออะไร

### 02 — Glossary + Data Prep
- คำศัพท์ทั้งหมด (RAG, LLM, Token, Embedding, Vector, etc.)
- ขั้นตอนเตรียม Data เข้า Vector DB
- Chunking พื้นฐาน

### 03 — Plan v2
- สิ่งที่เปลี่ยนจาก v1
- Architecture ใหม่
- Performance Optimization

### 04 — Architecture Diagrams
- High-Level Architecture
- Component Diagrams
- Data Flow Diagrams
- Sequence Diagrams

### 05 — Implementation Guide
- วิธีติดตั้ง
- วิธี Config
- วิธี Deploy
- ทีละขั้นตอน

### 06 — Chunking Deep Dive
- ทุกวิธี Chunking
- เปรียบเทียบ Performance
- เมื่อไหร่ใช้วิธีไหน

### 07 — Integration Guide
- ต่อกับอะไรได้บ้าง
- Use Case นอกเหนือจาก Chat
- Integration Patterns

### 08 — Client Channels
- API
- Webhook
- SDK (Python, JavaScript, Go, etc.)
- Streaming vs Non-streaming

## Quick Reference

| อยากทำ... | ไปดู |
|------------|-------|
| เริ่มต้นเรียนรู้ RAG | 01 |
| เข้าใจคำศัพท์ | 02 |
| เริ่ม Implement ครั้งแรก | 05 |
| เรียก API ใช้งาน | 09 |
| ดู Diagram Architecture | 04 |
| ต่อกับ LINE/Slack/Email | 07, 08 |
| Optimize Performance | 06 |
| Debug ปัญหา | 11 |
| ทดสอบคุณภาพระบบ | 10 |
| Hardening Security | 12 |
| เข้าใจ Architecture ลึกๆ | 03 |

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG — Retrieval-Augmented Generation]]
- [[wiki/concepts/llm-large-language-model|LLM — Large Language Model]]
- [[wiki/concepts/vector-database|Vector Database]]
- [[wiki/concepts/embedding|Embedding]]
- [[wiki/concepts/ai-agent|AI Agent]]
