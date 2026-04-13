---
title: "OTel Baggage"
type: concept
tags: [opentelemetry, baggage, w3c, context-propagation, metadata, multi-tenant]
sources: [Distributed Tracing & Context Propagation.md, เข้าใจ SetTextMapPropagator ทุกชิ้น — ทำไม ทำอะไร เชื่อมกันยังไง.md]
related: [wiki/concepts/context-propagation.md, wiki/concepts/w3c-trace-context.md, wiki/concepts/distributed-tracing.md]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

OTel Baggage (W3C Baggage standard) คือกลไกส่ง key-value metadata ข้าม service โดยอัตโนมัติผ่าน HTTP header `baggage` — ต่างจาก Span Attribute ตรงที่ service อื่นดึงค่าออกมาใช้ใน code ได้โดยตรง

## อธิบาย

Baggage แก้ปัญหาเมื่อต้องการส่งข้อมูลเช่น `tenantId`, `userId` จาก service แรก (API Gateway) ไปให้ทุก service ในสาย chain — โดยไม่ต้องแก้ function signature ทุกชั้น และไม่ต้องส่งใน request body

```
baggage: userId=user_789,tenantId=tenant_abc,region=asia
         key=value         key=value          key=value (คั่นด้วย comma)
```

### วิธีใช้ใน Go

```go
// Service A: ใส่ข้อมูลลง baggage
member, _ := baggage.NewMember("tenantId", "tenant_abc")
bag, _ := baggage.New(member)
ctx = baggage.ContextWithBaggage(ctx, bag)
// → otelgrpc inject header: baggage: tenantId=tenant_abc

// Service B, C, D รับมาอัตโนมัติผ่าน propagator
bag := baggage.FromContext(ctx)
tenantId := bag.Member("tenantId").Value()   // "tenant_abc"
```

## ประเด็นสำคัญ

- Baggage ต่างจาก TraceContext ตรงที่ TraceContext ส่ง "ตัวตน" ของ trace (TraceID, SpanID) ส่วน Baggage ส่ง "ข้อมูล" เพิ่มเติม
- **Baggage vs Span Attribute**: Baggage อยู่ใน header → service อื่นดึงใช้ใน code ได้; Attribute อยู่ใน Span → เห็นใน Jaeger UI แต่ service อื่นไม่รู้

| Feature | Baggage | Span Attribute |
|---|---|---|
| เก็บที่ไหน | HTTP Header | ใน Span (Jaeger เท่านั้น) |
| ส่งข้าม service | ✅ ส่งทุก service | ❌ ไม่ส่ง |
| เห็นใน Jaeger | ❌ (ต้อง set เอง) | ✅ เสมอ |
| ใช้ใน business logic | ✅ ดึงมาใช้ใน code ได้ | ❌ ดูได้แค่ใน Jaeger |
| ขนาด | จำกัด (~8KB header) | ใหญ่ได้มากกว่า |

- **กฎง่ายๆ**: "service อื่นต้องรู้ค่านี้เพื่อทำงานต่อไหม?" → ใช่ = Baggage, ไม่ = Span Attribute
- `propagation.Baggage{}` เป็น **optional** — ถ้าไม่ใส่ใน Composite propagator ยังทำ trace linking ได้ปกติ แค่ไม่รองรับ baggage header
- ควรใส่ `Baggage{}` ไว้เผื่อ เพราะ zero cost ถ้าไม่ใช้ (struct เปล่า, return ทันทีถ้าไม่มี baggage)

## ตัวอย่าง / กรณีศึกษา

**Multi-tenant system**: API Gateway รู้ `tenantId` จาก JWT → ใส่ใน baggage → ทุก service กรอง data ตาม tenant โดยไม่ต้องแก้ gRPC request body หรือ function signature

```go
// API Gateway authMiddleware:
member, _ := baggage.NewMember("tenantId", claims.TenantID)
bag, _ := baggage.New(member)
ctx := baggage.ContextWithBaggage(c.UserContext(), bag)

// Order Service repository:
bag := baggage.FromContext(ctx)
tenantId := bag.Member("tenantId").Value()
return r.db.Where("tenant_id = ?", tenantId).Find(&[]Order{})
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/context-propagation|Context Propagation]] — Baggage เป็น propagator อีกตัวที่ทำงานคู่กับ TraceContext ใน Composite propagator
- [[wiki/concepts/w3c-trace-context|W3C TraceContext]] — propagator สำหรับ trace identity (TraceID/SpanID) ส่วน Baggage สำหรับ metadata เพิ่มเติม

## แหล่งที่มา

- [[wiki/sources/distributed-tracing-context-propagation|Distributed Tracing & Context Propagation]]
- [[wiki/sources/settextmappropagator-explained|SetTextMapPropagator Explained]]
