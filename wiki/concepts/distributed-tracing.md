---
title: "Distributed Tracing"
type: concept
tags: [distributed-tracing, observability, microservices, jaeger, opentelemetry]
sources: [Distributed Tracing & Context Propagation.md, OpenTelemetry — คู่มือฉบับ Deep Dive.md]
related: [wiki/concepts/context-propagation.md, wiki/concepts/opentelemetry.md, wiki/concepts/w3c-trace-context.md]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

Distributed Tracing คือระบบติดตาม request เดียวข้ามหลาย service ใน Microservices — เหมือนติด GPS tracker ไว้กับ request ตั้งแต่ต้นจนจบ ทุก service บันทึกว่า "รับ request นี้ไว้ ใช้เวลา X ms"

## อธิบาย

ใน Monolith ถ้า error เกิดดู log ใน process เดียวได้เลย แต่ใน Microservices request 1 ใบทำให้เกิด log กระจายอยู่ใน 4–5 service คนละ process คนละ machine คนละ log file — ปัญหาคือไม่รู้ว่า log พวกนั้นเป็นของ request เดียวกันไหม

Distributed Tracing แก้ปัญหานี้โดยให้ request "พกตัวตน" (TraceID) ไปทุก service ที่ผ่าน แต่ละ service บันทึก span ของตัวเองพร้อมอ้างอิง TraceID เดิม — Jaeger/Grafana Tempo รวม spans ทั้งหมดเป็น waterfall view

## ประเด็นสำคัญ

| คำศัพท์ | ความหมาย | ตัวอย่าง |
|---|---|---|
| **Trace** | การเดินทางทั้งหมดของ 1 request | TraceID: `b71df886bfbb3cca` |
| **Span** | งานชิ้นเดียวใน 1 service | SpanID: `9c4a6dbf93d0594d` |
| **Parent Span** | span ที่เรียก span อื่น | Central Control เรียก Role Permission |
| **Child Span** | span ที่ถูกเรียก | Role Permission ถูกเรียกจาก Central Control |
| **TraceID** | เลข ID ของ trace ทั้งหมด | เหมือนเลขพัสดุ |
| **SpanID** | เลข ID ของ span นั้นๆ | เหมือนเลข tracking ของแต่ละด่าน |

Span แต่ละตัวมี fields: SpanID, ParentSpanID, Name, StartTime, EndTime, Attributes, Events, Status, Links

## ตัวอย่าง / กรณีศึกษา

```
Trace: b71df886bfbb3cca                                  total: 57ms
│
├── [Central Control]      ListRoles    ████████████████  45ms
│   SpanID: 9c4a6dbf
│   │
│   └── [Role Permission]  ListRoles       ████████       12ms
│       SpanID: fa3c19e8
│       ParentSpanID: 9c4a6dbf
│       │
│       └── [PostgreSQL]   SELECT roles       ████         5ms
│           SpanID: bb2d11a3
│           ParentSpanID: fa3c19e8
```

เห็น waterfall → รู้ทันทีว่าช้าที่ service ไหน layer ไหน

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/context-propagation|Context Propagation]] — กลไกส่ง TraceID/SpanID ข้าม service ผ่าน network header
- [[wiki/concepts/opentelemetry|OpenTelemetry]] — framework ที่ implement distributed tracing (และ metrics, logs)
- [[wiki/concepts/w3c-trace-context|W3C TraceContext]] — standard format ของ `traceparent` header ที่ใช้ส่ง TraceID

## แหล่งที่มา

- [[wiki/sources/distributed-tracing-context-propagation|Distributed Tracing & Context Propagation]]
- [[wiki/sources/opentelemetry-deep-dive|OpenTelemetry Deep Dive]]
