---
title: "เข้าใจ SetTextMapPropagator ทุกชิ้น — ทำไม ทำอะไร เชื่อมกันยังไง"
type: source
source_file: raw/notes/otel/เข้าใจ SetTextMapPropagator ทุกชิ้น — ทำไม ทำอะไร เชื่อมกันยังไง.md
url: ""
published: 2026-04-13
tags: [opentelemetry, textmappropagator, tracecontext, baggage, go]
related: [wiki/concepts/context-propagation.md, wiki/concepts/w3c-trace-context.md, wiki/concepts/otel-baggage.md, wiki/concepts/otel-go-instrumentation.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/observability/เข้าใจ SetTextMapPropagator ทุกชิ้น — ทำไม ทำอะไร เชื่อมกันยังไง.md|Original file]]

## สรุป

อธิบายทุกส่วนของ `otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(...))` ว่าแต่ละชิ้นทำอะไร — ทำไม global registry ถึงสำคัญ, Composite propagator รวมหลายตัวอย่างไร, TraceContext{} inject/extract traceparent header อย่างไร, Baggage{} ต่างจาก Span Attribute อย่างไร พร้อม end-to-end flow จริง (Browser → API → gRPC service) และตารางเปรียบเทียบ Baggage vs Span Attribute

## ประเด็นสำคัญ

- **`otel.SetTextMapPropagator`** = ลงทะเบียน global propagator ที่ library ทุกตัว (otelfiber, otelgrpc, otelhttp) ดึงไปใช้เองผ่าน `otel.GetTextMapPropagator()`
- **ถ้าไม่ Set** → library ได้ noop propagator → ไม่มีการ inject/extract → trace chain ขาด
- **`NewCompositeTextMapPropagator`** = รวมหลาย propagator ให้ทำงานพร้อมกัน (Inject ทีละตัวเรียงลำดับ, Extract ผ่าน chain ทุกตัว)
- **`TraceContext{}`**: implement W3C TraceContext — Inject แปลง TraceID+SpanID → `traceparent` header; Extract แปลง `traceparent` header → SpanContext ใน ctx; **ขาดไม่ได้**
- **`Baggage{}`**: implement W3C Baggage — ส่ง key-value metadata ข้าม service ผ่าน `baggage` header; **optional แต่ควรใส่ไว้เผื่อ** เพราะ zero cost ถ้าไม่ใช้
- **Baggage vs Span Attribute**: Baggage เก็บใน header → service อื่นอ่านใน code ได้; Attribute เก็บใน Span → เห็นใน Jaeger แต่ service อื่นไม่รู้
- **กฎง่ายๆ**: "service อื่นต้องรู้ค่านี้เพื่อทำงานต่อไหม?" → ใช่ = Baggage, ไม่ = Span Attribute

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- ตัวอย่าง use case ของ Baggage ที่ดี: Multi-tenant system ใส่ `tenantId` ที่ API Gateway → ทุก service กรอง data ตาม tenant ได้โดยไม่ต้องแก้ function signature
- `TraceContext{}.Extract`: parse traceparent → สร้าง SpanContext{Remote: true} → ตอน tracer.Start ใช้เป็น parent span ทำให้ traceId เดิมใช้ต่อ ไม่สร้างใหม่
- ลบ `TraceContext{}` ออก → `traceparent` ไม่ถูกอ่าน/เขียน → trace chain ขาดทุก service hop → **อย่าทำ**
- ลบ `Baggage{}` ออก → trace ยังทำงาน แต่ baggage header ถูกละเลย → ถ้าไม่ใช้ baggage เลย ไม่มีผล

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/context-propagation|Context Propagation]]
- [[wiki/concepts/w3c-trace-context|W3C TraceContext]]
- [[wiki/concepts/otel-baggage|OTel Baggage]]
- [[wiki/concepts/otel-go-instrumentation|OTel Go Instrumentation]]
