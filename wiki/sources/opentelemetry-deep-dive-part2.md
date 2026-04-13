---
title: "OpenTelemetry — Deep Dive ฉบับ Part 2"
type: source
source_file: raw/notes/otel/OpenTelemetry — Deep Dive ฉบับ Part 2.md
url: ""
published: 2026-04-13
tags: [opentelemetry, textmapcarrier, textmappropagator, jaeger, batchspanprocessor, go]
related: [wiki/concepts/context-propagation.md, wiki/concepts/opentelemetry.md, wiki/concepts/distributed-tracing.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/otel/OpenTelemetry — Deep Dive ฉบับ Part 2.md|Original file]]

## สรุป

Deep Dive Part 2 เจาะลึก internals ของ OTel — แยก role ของ TextMapCarrier (กล่องเก็บ key-value header) กับ TextMapPropagator (logic อ่านเขียน context), วิธีที่ spans ถูกส่ง (BatchSpanProcessor buffer แล้วส่งเป็น batch), วิธี Jaeger รวม spans จากหลาย service ที่มาไม่พร้อมกัน และ Timeline จริงตั้งแต่ request เข้ามาจนเห็น trace ใน Jaeger UI พร้อมตัวอย่างข้อมูลจริงทุกขั้นตอน

## ประเด็นสำคัญ

- **TextMapCarrier** = interface ที่เป็น abstraction ของ "สิ่งที่เก็บ key-value string" — มี 3 methods: `Get(key)`, `Set(key, value)`, `Keys()` — ตัวเองไม่รู้ว่าข้างในคืออะไร แค่ Get/Set ได้
- **Implementation ของ TextMapCarrier**: `HeaderCarrier` (ครอบ `http.Header`), `MapCarrier` (ครอบ `map[string]string` — ใช้กับ Kafka, gRPC metadata, tests)
- **TextMapPropagator** = logic ที่รู้วิธีอ่านเขียน context — TraceContext{} รู้ format ของ `traceparent`, Baggage{} รู้ format ของ `baggage`
- **แยก role**: Carrier รู้ว่า "เก็บข้อมูลที่ไหน", Propagator รู้ว่า "เขียนข้อมูลอะไร อย่างไร"
- **BatchSpanProcessor**: spans ไม่ได้ส่งทันทีเมื่อ end — buffer ไว้ส่งเป็น batch (ลด network overhead) ตาม interval หรือเมื่อ buffer เต็ม
- **Jaeger assembly**: Jaeger รวม spans โดยใช้ TraceID เป็น key — spans จากหลาย service มาถึงต่างเวลาก็รวมได้ ไม่ต้องมาพร้อมกัน

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- `HeaderCarrier` ใช้ `http.Header.Get()` ซึ่ง case-insensitive ตาม HTTP spec
- `MapCarrier` case-sensitive — ต้องระวังเรื่อง key capitalization เมื่อใช้กับ Kafka/gRPC
- Spans ส่งผ่าน OTLP (OpenTelemetry Protocol) ซึ่งเป็น protobuf-based binary format
- ทั้งสอง Carrier type มีอยู่ใน SDK แล้ว: `go.opentelemetry.io/otel/propagation`

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/context-propagation|Context Propagation]]
- [[wiki/concepts/opentelemetry|OpenTelemetry]]
- [[wiki/concepts/distributed-tracing|Distributed Tracing]]
