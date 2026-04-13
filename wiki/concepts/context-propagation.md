---
title: "Context Propagation"
type: concept
tags: [context-propagation, opentelemetry, textmappropagator, textmapcarrier, w3c]
sources: [Distributed Tracing & Context Propagation.md, OpenTelemetry — Deep Dive ฉบับ Part 2.md, เข้าใจ SetTextMapPropagator ทุกชิ้น — ทำไม ทำอะไร เชื่อมกันยังไง.md]
related: [wiki/concepts/distributed-tracing.md, wiki/concepts/w3c-trace-context.md, wiki/concepts/otel-baggage.md, wiki/concepts/opentelemetry.md]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

Context Propagation คือกลไกส่ง "ตัวตน" ของ request (TraceID/SpanID) ข้าม service ผ่าน network header — เพราะแต่ละ service อยู่คนละ process ไม่แชร์ memory กัน จึงต้องส่งผ่าน header เท่านั้น

## อธิบาย

แต่ละ service อยู่คนละ process → memory ไม่แชร์กัน → TraceID ที่อยู่ใน context ของ service A จะไม่ไปถึง service B โดยอัตโนมัติ

Context Propagation แก้ปัญหานี้ด้วยกลไก **Inject/Extract**:
- **Inject** = อ่าน TraceID/SpanID จาก context ใน memory → เขียนลง network header ก่อนส่ง request
- **Extract** = อ่าน network header ที่รับมา → สร้าง context ขึ้นใหม่ใน memory พร้อม TraceID/SpanID จาก caller

### สองส่วนหลัก

**TextMapPropagator** = "logic" ที่รู้วิธีอ่านเขียน context — บอกว่า "เขียนข้อมูลลง field อะไร ในรูปแบบไหน"

```go
type TextMapPropagator interface {
    Inject(ctx context.Context, carrier TextMapCarrier)
    Extract(ctx context.Context, carrier TextMapCarrier) context.Context
    Fields() []string // บอกว่าใช้ header อะไรบ้าง
}
```

**TextMapCarrier** = "กล่อง" ที่เก็บ key-value string — abstraction ของ HTTP headers / gRPC metadata

```go
type TextMapCarrier interface {
    Get(key string) string
    Set(key string, value string)
    Keys() []string
}
```

- `HeaderCarrier` — ครอบ `http.Header` (case-insensitive ตาม HTTP spec)
- `MapCarrier` — ครอบ `map[string]string` (ใช้กับ Kafka headers, gRPC metadata, tests)

## ประเด็นสำคัญ

- Propagator ไม่เกี่ยวกับ transport layer (gRPC vs HTTP vs Kafka) — มันแค่บอกว่า "เขียนลง field อะไร"
- **Global propagator** ต้อง set ด้วย `otel.SetTextMapPropagator(...)` — library เช่น otelgrpc, otelhttp, otelfiber ดึงไปใช้เองผ่าน `otel.GetTextMapPropagator()`
- ถ้าไม่ Set → ได้ noop propagator → ไม่มีการ inject/extract → trace ขาด
- Propagator format ต้องตรงกันทั้ง chain (ถ้า service A ส่ง W3C แต่ service B อ่าน B3 → extract ไม่ได้)

## ตัวอย่าง / กรณีศึกษา

```
Central Control (TraceID: abc, SpanID: 111)
    ↓ otelgrpc.NewClientHandler() Inject
    traceparent: 00-abc-111-01   →→→→→→→→ network header
                                           ↓
                                     Role Permission
                                     otelgrpc.NewServerHandler() Extract
                                     SpanContext{TraceID: abc, ParentID: 111}
                                     สร้าง child span → SpanID: 222
                                     TraceID เดิม!
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/distributed-tracing|Distributed Tracing]] — context propagation คือกลไกที่ทำให้ distributed tracing ทำงานได้
- [[wiki/concepts/w3c-trace-context|W3C TraceContext]] — propagator ที่ใช้ `traceparent` header (W3C standard)
- [[wiki/concepts/otel-baggage|OTel Baggage]] — propagator อีกตัวที่ส่ง metadata ข้าม service ผ่าน `baggage` header
- [[wiki/concepts/otel-go-instrumentation|OTel Go Instrumentation]] — วิธีติดตั้ง propagator ใน Go services

## แหล่งที่มา

- [[wiki/sources/distributed-tracing-context-propagation|Distributed Tracing & Context Propagation]]
- [[wiki/sources/opentelemetry-deep-dive-part2|OpenTelemetry Deep Dive Part 2]]
- [[wiki/sources/settextmappropagator-explained|SetTextMapPropagator Explained]]
