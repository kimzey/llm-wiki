---
title: "c.UserContext() vs getSpanContext(c) — Fiber OTel Context Bug"
type: source
source_file: raw/notes/otel/context-usercontext-vs-getspancontext.md
url: ""
published: 2026-04-13
tags: [fiber, opentelemetry, otelfiber, context, bug, go]
related: [wiki/concepts/otel-go-instrumentation.md, wiki/concepts/context-propagation.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/otel/context-usercontext-vs-getspancontext.md|Original file]]

## สรุป

อธิบาย bug ใน Fiber framework ที่ function `getSpanContext(c)` (ถูกลบไปแล้ว) ทำให้ OTel trace ไม่ propagate ถึง use case/repository เลย — root cause คือ Fiber มี context สองชั้น (fasthttp RequestCtx กับ Go standard context) และ code เดิมอ่านผิด key + ผิด type จาก Locals() จนต้อง fallback ไปหา fasthttp context ที่ไม่มี span ของ otelfiber

## ประเด็นสำคัญ

- **Fiber มี context 2 แบบที่ต่างกันสิ้นเชิง**:
  - `c.Context()` → `*fasthttp.RequestCtx` — HTTP low-level, ไม่มี OTel span, ใช้ดึงข้อมูล HTTP
  - `c.UserContext()` → `context.Context` — Go standard, middleware inject span ได้ผ่าน `c.SetUserContext(ctx)`, ใช้กับ tracing
- **otelfiber.Middleware()** inject span เข้า `c.UserContext()` ผ่าน `c.SetUserContext(ctx)` ก่อนเรียก handler
- **Bug ใน `getSpanContext(c)`** มี 2 ชั้น: (1) อ่าน Locals ผิด key (`"fiber-otel-tracer"` แต่ otelfiber ใช้ `"gofiber-contrib-tracer-fiber"`), (2) type ที่ otelfiber เก็บคือ `trace.Tracer` ไม่ใช่ `context.Context` → type assertion fail เสมอ → fallback ไป `c.Context()` (fasthttp) ที่ไม่มี span
- **ผลลัพธ์**: ทุก handler ได้ fasthttp context ธรรมดา — span ไม่เคย propagate ถึง use case / repository เลย
- **วิธีแก้**: ใช้ `c.UserContext()` แทนทุกที่

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- Source จริงของ otelfiber v2.2.3: บรรทัดสำคัญคือ `c.SetUserContext(ctx)` ที่ inject span ลง UserContext
- Ticket อ้างอิง: `enhance/1961-fix-tracer`

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/otel-go-instrumentation|OTel Go Instrumentation]]
- [[wiki/concepts/context-propagation|Context Propagation]]
