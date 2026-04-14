---
title: "Agent Systems — ระบบ AI Agent ฉบับสมบูรณ์"
type: source
source_file: "raw/notes/ai-engineering/03_agent_systems.md"
tags: [agent, multi-agent, react, planning, tool-calling, human-in-the-loop]
related: [wiki/concepts/ai-agent, wiki/concepts/tool-calling, wiki/concepts/ai-agents-system]
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../../raw/notes/ai-engineering/03_agent_systems.md|Original file]]

## สรุป

คู่มือ Agent Systems ครอบคลุม Agent Loop, 4 ประเภท Architecture (ReAct, Plan-and-Execute, Reflection, Self-Consistency), Multi-Agent Patterns (Supervisor-Worker, Pipeline, Debate), Tool Design Best Practices, Human-in-the-Loop, State Management, และ Observability สำหรับระบบ Agent ใน production

## ประเด็นสำคัญ

- **Agent Loop**: รับ state → ตัดสินใจ (tool/ask/answer) → execute → รับผล → repeat
- **ReAct**: Thought → Action → Observation สลับกันจนได้คำตอบ (pattern ที่นิยมที่สุด)
- **Plan-and-Execute**: วางแผนก่อนทั้งหมด แล้วจึง execute — ยืดหยุ่นน้อยกว่า แต่ตรวจสอบง่าย
- **Self-Consistency**: รันหลายรอบ เลือก majority vote — ดีสำหรับโจทย์คณิตศาสตร์
- **Multi-Agent Supervisor-Worker**: Supervisor วิเคราะห์งาน → มอบหมายให้ Worker แบบขนาน → รวมผล
- **Tool Design**: แต่ละ tool ทำหน้าที่เดียว, description ชัดเจน, error handling ครบ
- **Risk Levels**: Read-only (ทำได้เลย) → Reversible (log ไว้) → Irreversible (ต้องขอ approval)
- **Observability**: log trace_id, tool calls, tokens, latency ทุก step

## ข้อมูล / หลักฐาน ที่น่าสนใจ

> [!note] Memory Types ของ Agent
> | ประเภท | ที่เก็บ | ตัวอย่าง |
> |--------|---------|----------|
> | In-Context | Context window ปัจจุบัน | ประวัติสนทนา |
> | Short-term External | Redis | สถานะ task |
> | Long-term External | Vector DB, SQL | ความรู้, ประวัติผู้ใช้ |
> | Episodic | บันทึก | "ครั้งก่อนผู้ใช้ชอบ..." |

- Checkpoint Pattern: บันทึกจุด checkpoint ทุก step — ถ้า crash กลางทางสามารถ resume ได้
- Metrics ที่ตรวจ: Task Success Rate, Tool Call Count, End-to-end Latency, Human Intervention Rate
- Customer Service Agent ตัวอย่าง: ตรวจออเดอร์ → ประเมินสิทธิ์คืนเงิน → คืนเงินอัตโนมัติ

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/ai-agent|AI Agent]]
- [[wiki/concepts/tool-calling|Tool Calling]]
- [[wiki/concepts/ai-agents-system|AI Agents System]]
- [[wiki/concepts/agentic-rag|Agentic RAG]]
