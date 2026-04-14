---
title: "Cost Optimization — ลดต้นทุน AI"
type: source
source_file: "raw/notes/ai-engineering/10_cost_optimization.md"
tags: [cost-optimization, caching, model-routing, batch-processing, token-optimization]
related: [wiki/concepts/ai-cost-optimization, wiki/concepts/semantic-caching, wiki/concepts/ai-production-deployment]
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../../raw/notes/ai-engineering/10_cost_optimization.md|Original file]]

## สรุป

คู่มือ Cost Optimization ครอบคลุมการคำนวณต้นทุน (LLM + Infrastructure + Development), LLM Routing (ใช้โมเดลให้เหมาะงาน), Multi-level Caching, Token Optimization, Batch Processing (ราคาถูกกว่า 50%), Architecture Patterns (Async Queue, Streaming SSE, RAG with Preprocessing), และ Roadmap ประหยัด 30-90%

## ประเด็นสำคัญ

- **Cost Components**: Input tokens + Output tokens + Embedding tokens + Fine-tuning
- **LLM Router**: จัดระดับ complexity → trivial/simple → Haiku, moderate/complex → Sonnet, expert → Opus
- **การประหยัดจาก Routing**: ก่อน $675/เดือน (Sonnet ทั้งหมด) → หลัง ~$250 (60% Haiku + 35% Sonnet) = ประหยัด 63%
- **Prompt Caching**: cache prefix ที่ไม่เปลี่ยน → ประหยัด 90% บน cached tokens = ลด cost 81%
- **Batch API**: ถูกกว่า 50% แต่ใช้เวลา 1-24 ชั่วโมง — เหมาะงาน async
- **Context Compression**: summarize history เก่า → ลด input tokens ต่อ request
- **Concise Instructions**: เพิ่มใน system prompt → ประหยัด output 20-40%

## ข้อมูล / หลักฐาน ที่น่าสนใจ

> [!note] Cost Optimization Roadmap
> - **Phase 1** (ทำได้ทันที): Exact-match cache + limit output + ลด history → ประหยัด 30-50%
> - **Phase 2** (1-2 สัปดาห์): Semantic cache + LLM Router + Prompt Caching → ประหยัด 50-70%
> - **Phase 3** (1-3 เดือน): Batch processing + Fine-tune small model + Self-hosted embedding → ประหยัด 70-90%

```python
# Prompt Caching (Anthropic)
{"type": "text", "text": system_doc, "cache_control": {"type": "ephemeral"}}
# 100k tokens: $0.30 → ด้วย cache: $0.057 (ประหยัด 81%!)
```

- Architecture Decision: Real-time Chat → Streaming+Sonnet, Batch Classification → BatchAPI+Haiku, High Volume Simple → Haiku+Batching+Cache

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/ai-cost-optimization|การลดต้นทุน AI]]
- [[wiki/concepts/semantic-caching|Semantic Caching]]
- [[wiki/concepts/ai-production-deployment|การ Deploy ระบบ AI สู่ Production]]
