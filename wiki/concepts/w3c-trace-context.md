---
title: "W3C TraceContext"
type: concept
tags: [w3c, traceparent, tracestate, opentelemetry, distributed-tracing, standard]
sources: [Distributed Tracing & Context Propagation.md, เข้าใจ SetTextMapPropagator ทุกชิ้น — ทำไม ทำอะไร เชื่อมกันยังไง.md, OpenTelemetry — คู่มือฉบับ Deep Dive.md]
related: [wiki/concepts/context-propagation.md, wiki/concepts/distributed-tracing.md, wiki/concepts/otel-baggage.md]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

W3C TraceContext (RFC W3C 2020) คือ standard สากลสำหรับส่ง trace context ข้าม service ผ่าน HTTP header `traceparent` — รองรับโดย Jaeger, Grafana Tempo, Datadog, Zipkin, AWS X-Ray ทุกตัว

## อธิบาย

W3C TraceContext กำหนด format ของ header 2 ตัว:
- **`traceparent`** — บรรจุ TraceID, SpanID, และ sampling flags
- **`tracestate`** — optional, vendor-specific data

### รูปแบบ traceparent header

```
traceparent: 00-b71df886bfbb3cca29211a10b74c6212-9c4a6dbf93d0594d-01
             │  │                                │                │
             │  │                                │                └── flags (01=sampled, 00=not sampled)
             │  │                                └── Parent SpanID (64-bit hex = 16 chars)
             │  └── TraceID (128-bit hex = 32 chars)
             └── version (00 = current version)
```

### กระบวนการ Inject (TraceContext{}.Inject)

1. ดึง SpanContext จาก context ใน memory
2. แปลง TraceID bytes → hex string (32 chars)
3. แปลง SpanID bytes → hex string (16 chars)
4. เช็ค sampled flag
5. สร้าง string รูปแบบ `00-<traceId>-<spanId>-<flags>`
6. เรียก `carrier.Set("traceparent", string)` → เขียนลง header

### กระบวนการ Extract (TraceContext{}.Extract)

1. เรียก `carrier.Get("traceparent")`
2. split ด้วย `-` → [version, traceId, spanId, flags]
3. parse TraceID → [16]byte
4. parse SpanID → [8]byte (กลายเป็น ParentSpanID ของ child span)
5. parse flags → TraceFlags
6. สร้าง `SpanContext{TraceID, SpanID, Flags, Remote: true}`
7. ฝัง SpanContext ลงใน ctx → ตอน `tracer.Start(ctx, ...)` span ใหม่จะใช้ TraceID เดิม

## ประเด็นสำคัญ

- **`Remote: true`** ใน SpanContext บอกว่า span นี้มาจากภายนอก (caller)
- **flags `01`** = sampled → ส่งไป backend; **`00`** = ไม่ส่ง (แต่ยังสร้าง trace context)
- W3C เป็น standard ปัจจุบัน ดีกว่า B3 (Zipkin), Jaeger (`uber-trace-id`), AWS X-Ray (`X-Amzn-Trace-Id`) ในแง่ interoperability
- ทั้ง chain ต้องใช้ format เดียวกัน — W3C → W3C ✅, W3C → B3 ❌ (extract ไม่ได้)

## ตัวอย่าง / กรณีศึกษา

ตัวอย่างจากโปรเจกต์จริง:
```
// Central Control ส่งมา:
traceparent: 00-b71df886bfbb3cca29211a10b74c6212-9c4a6dbf93d0594d-01

// Role Permission อ่านแล้วสร้าง context:
TraceID:      b71df886bfbb3cca29211a10b74c6212  ← เอาจาก header
ParentSpanID: 9c4a6dbf93d0594d                  ← SpanID ของ Central Control
SpanID:       fa3c19e820b74a12                  ← สร้างใหม่ (child span)
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/context-propagation|Context Propagation]] — W3C TraceContext คือ implementation ของ TextMapPropagator สำหรับ trace linking
- [[wiki/concepts/otel-baggage|OTel Baggage]] — propagator อีกตัวที่ส่ง metadata (ไม่ใช่ trace identity) ผ่าน `baggage` header
- [[wiki/concepts/distributed-tracing|Distributed Tracing]] — W3C TraceContext คือ "ฉลาก" ที่ผูก spans ต่างๆ เข้าเป็น trace เดียว

## แหล่งที่มา

- [[wiki/sources/observability/distributed-tracing-context-propagation|Distributed Tracing & Context Propagation]]
- [[wiki/sources/observability/settextmappropagator-explained|SetTextMapPropagator Explained]]
- [[wiki/sources/observability/opentelemetry-deep-dive|OpenTelemetry Deep Dive]]
