---
title: "Haystack Deep Dive — Phase 6: API, Auth, Observability & Ecosystem"
type: source
source_file: raw/notes/Hetstack/haystack-phase6-api-auth-observability.md
url: ""
published: 2026-04-01
tags: [haystack, api, auth, jwt, observability, langfuse, opentelemetry, prometheus, grafana, hayhooks]
related: [wiki/sources/haystack-phase5-production.md, wiki/concepts/haystack-framework.md, wiki/concepts/opentelemetry.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/frameworks/haystack/haystack-phase6-api-auth-observability.md|Original file]]

## สรุป

Phase 6 ครอบคลุม Production Stack ครบวงจร — Hayhooks (official API serving tool), FastAPI với JWT/API-key/RBAC, Observability ด้วย Langfuse/OpenTelemetry/Arize Phoenix, Monitoring ด้วย Prometheus+Grafana, Rate Limiting, Guardrails, และ Haystack vs LangChain ecosystem comparison

## ประเด็นสำคัญ

- **Hayhooks**: official tool จาก deepset — serialize pipeline เป็น YAML แล้ว serve ทันที, auto-generate REST API
- **Langfuse** คือ tool กลางที่ใช้ได้ทั้ง Haystack และ LangChain — เพิ่มด้วย `LangfuseConnector` 1 บรรทัด
- **OpenTelemetry tracing** รองรับ built-in — `haystack.tracing.enable_tracing(OpenTelemetryTracer())` → ส่งไป Jaeger/Tempo
- **Arize Phoenix**: local, free, zero-config observability — `HaystackInstrumentor().instrument()`
- **Input/Output Guardrails**: Custom Component ตรวจ prompt injection + PII ใน response
- **Haystack ชนะ LangChain** เรื่อง Pipeline type-safety, debug ง่ายกว่า — แต่ LangChain ง่ายกว่าเรื่อง API serving out-of-box (LangServe)

## ข้อมูล / หลักฐาน ที่น่าสนใจ

**Recommended Production Stack (2025):**
```
Pipeline:  Haystack 2.x
Vector DB: Qdrant (self-host) หรือ Qdrant Cloud
API:       FastAPI + Hayhooks
Auth:      JWT (python-jose)
Tracing:   Langfuse (self-host)
Metrics:   Prometheus + Grafana
Deploy:    Docker Compose → Kubernetes
Eval:      RAGAS + Langfuse Evals
```

**Langfuse tracks อัตโนมัติ:**
- Input query, Retrieved documents + scores
- Prompt ที่ส่งไป LLM, Response + token count + cost
- Latency ทั้ง pipeline, User feedback

**Haystack vs LangChain ตรงๆ:**
```
LangChain: Ecosystem ครบกว่า out-of-box (LangServe, LangSmith)
Haystack:  Pipeline แม่นยำกว่า, type-safe, production-grade
Langfuse:  ใช้ได้ทั้งคู่ — ไม่ต้องเลือก
```

**Rate Limiting ด้วย slowapi:** `@limiter.limit("20/minute")`

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่มีข้อขัดแย้ง — ข้อมูล OTel ใน phase 6 สอดคล้องกับ OTel concepts ที่มีอยู่ใน wiki

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/haystack-framework|Haystack Framework]]
- [[wiki/concepts/opentelemetry|OpenTelemetry]]
- [[wiki/concepts/rag-evaluation|RAG Evaluation]]
