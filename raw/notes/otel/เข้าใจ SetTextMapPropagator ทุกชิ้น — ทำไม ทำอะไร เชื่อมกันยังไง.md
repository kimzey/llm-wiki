# เข้าใจ SetTextMapPropagator ทุกชิ้น — ทำไม ทำอะไร เชื่อมกันยังไง

> อธิบายจากพื้นฐานที่สุด ไปจนถึงเห็นภาพรวมทั้งหมด

---

## ก่อนอ่าน — โค้ดที่งงอยู่คือนี้

```go
otel.SetTextMapPropagator(
    propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ),
)
```

บทนี้จะอธิบาย **ทุกคำ** ในโค้ดนี้ว่าคืออะไร ทำอะไร และทำไมต้องมีทั้งสองตัว

---

## ปัญหาที่ต้องแก้ก่อน — เข้าใจ "ทำไม" ก่อน "ทำอะไร"

### สถานการณ์: ไม่มีโค้ดนี้เลย

```
Service A เรียก Service B ด้วย HTTP:

Service A:
  ctx มี traceId = "abc123"
  req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
  client.Do(req)
  // ← ส่ง request ออกไป แต่ header ว่างเปล่า!

Service B รับ request:
  header ว่าง → ไม่รู้ว่ามี traceId = "abc123" อยู่
  → สร้าง root span ใหม่ traceId = "xyz999"

Jaeger:
  Trace abc123: มีแค่ Service A
  Trace xyz999: มีแค่ Service B
  → แยกกันสนิท ดู trace ข้ามไม่ได้เลย
```

**โค้ด 4 บรรทัดข้างบนแก้ปัญหานี้** — มันทำให้ traceId วิ่งข้าม service ได้

---

## แยกทำความเข้าใจทีละชิ้น

---

### ชิ้นที่ 1: `otel.SetTextMapPropagator(...)`

```go
otel.SetTextMapPropagator(...)
```

#### คืออะไร?

**`otel`** = package กลางของ OpenTelemetry ที่เก็บ "global settings"
เหมือน config ที่ทั้ง app ใช้ร่วมกัน

**`SetTextMapPropagator`** = บอกระบบว่า "ถ้าจะส่ง/รับ trace context ผ่าน header ให้ใช้ตัวนี้"

```
Global Registry ของ OTEL:
  ┌─────────────────────────────────────────┐
  │  otel global                            │
  │  ─────────────────────────────────────  │
  │  TracerProvider  = tp          (set)    │
  │  TextMapPropagator = ???   ← ตรงนี้    │
  │  MeterProvider   = noopMeter  (default) │
  └─────────────────────────────────────────┘

หลัง SetTextMapPropagator:
  TextMapPropagator = CompositeTextMapPropagator{
      TraceContext{},
      Baggage{},
  }
```

#### ทำไม "Global"?

เพราะ library อื่นๆ เช่น otelfiber, otelgrpc จะเรียก `otel.GetTextMapPropagator()` เอง
ถ้าเราไม่ Set ไว้ → library เหล่านั้นจะได้ **no-op propagator** ที่ไม่ทำอะไรเลย

```go
// ภายใน otelfiber.Middleware() (source code จริง):
func Middleware() fiber.Handler {
    return func(c *fiber.Ctx) error {
        // เรียก Global Propagator ที่เรา Set ไว้
        propagator := otel.GetTextMapPropagator()
        ctx := propagator.Extract(c.Context(), ...)
        // ...
    }
}

// ถ้าเราไม่ Set → GetTextMapPropagator() คืน noop
// → Extract ไม่ทำอะไร → trace chain ขาด
```

---

### ชิ้นที่ 2: `propagation.NewCompositeTextMapPropagator(...)`

```go
propagation.NewCompositeTextMapPropagator(
    propagation.TraceContext{},
    propagation.Baggage{},
)
```

#### คืออะไร?

**Composite** = รวมหลายอย่างเป็นอันเดียว

ลองนึกภาพว่า propagator แต่ละตัวรับผิดชอบ header คนละตัว:
- `TraceContext{}` รับผิดชอบ `traceparent` header
- `Baggage{}` รับผิดชอบ `baggage` header

Composite = ทำงานทั้งสองพร้อมกันในคำสั่งเดียว

```
CompositeTextMapPropagator.Inject(ctx, carrier):
  → TraceContext{}.Inject(ctx, carrier)   เขียน traceparent
  → Baggage{}.Inject(ctx, carrier)        เขียน baggage

CompositeTextMapPropagator.Extract(ctx, carrier):
  → ctx = TraceContext{}.Extract(ctx, carrier)  อ่าน traceparent
  → ctx = Baggage{}.Extract(ctx, carrier)       อ่าน baggage
  → return ctx (มีทั้งสองอย่างแล้ว)
```

#### ถ้าใช้แค่ตัวเดียวจะเป็นยังไง?

```go
// ถ้าใส่แค่ TraceContext:
otel.SetTextMapPropagator(propagation.TraceContext{})
// → inject/extract แค่ traceparent header
// → baggage header ถูกละเลยทั้งหมด
// → tenantId, userId ที่ใส่ baggage จะไม่ถูกส่งข้าม service

// ถ้าใส่แค่ Baggage:
otel.SetTextMapPropagator(propagation.Baggage{})
// → inject/extract แค่ baggage header
// → traceparent header ไม่ถูกอ่าน/เขียน
// → trace chain ขาด!

// ต้องมีทั้งสอง:
otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
    propagation.TraceContext{},  // ← เชื่อม trace
    propagation.Baggage{},       // ← ส่ง metadata
))
```

---

### ชิ้นที่ 3: `propagation.TraceContext{}`

#### คืออะไร?

**TraceContext** = implementation ของ **W3C Trace Context Standard**

มันรู้วิธี:
- เขียน `traceId` + `spanId` ลงใน `traceparent` header
- อ่าน `traceparent` header กลับมาเป็น SpanContext

```
TraceContext{} รับผิดชอบ 2 headers:

  traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
  tracestate:  rojo=00f067aa0ba902b7  (optional, vendor-specific)
```

#### ทำอะไรตอน Inject?

```
input:  ctx ที่มี active span (traceId=abc, spanId=111)

กระบวนการ:
  1. ดึง SpanContext จาก ctx
  2. แปลง traceId bytes → hex string  "4bf92f35..."
  3. แปลง spanId bytes → hex string   "00f067aa..."
  4. เช็ค sampled flag → "01"
  5. สร้าง string: "00-4bf92f35...-00f067aa...-01"
  6. เรียก carrier.Set("traceparent", string นั้น)

output:  carrier มี header:
  traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

#### ทำอะไรตอน Extract?

```
input:  carrier ที่มี header:
  traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01

กระบวนการ:
  1. เรียก carrier.Get("traceparent")
  2. split ด้วย "-" → ["00", "4bf92f35...", "00f067aa...", "01"]
  3. parse traceId  → [16]byte
  4. parse spanId   → [8]byte
  5. parse flags    → TraceFlags(01) = sampled
  6. สร้าง SpanContext{
         TraceID: ...,
         SpanID:  ...,     ← นี่คือ parentSpanId ของ span ที่จะสร้างต่อ
         Flags:   sampled,
         Remote:  true,    ← บอกว่ามาจากภายนอก
     }
  7. ฝัง SpanContext ลงใน ctx ใหม่

output:  ctx ที่มี SpanContext พร้อมใช้
  → ตอน tracer.Start(ctx, ...) → span ใหม่จะใช้ traceId เดิม
  → และ parentSpanId = spanId ที่ extract มา
```

#### ผลลัพธ์ที่ได้

```
Service A (traceId=abc, spanId=111):
  inject → traceparent: 00-abc-111-01

Service B รับและ extract:
  → SpanContext{traceId=abc, spanId=111}
  → tracer.Start(ctx, "HandleOrder")
      → สร้าง Span{traceId=abc, spanId=222, parentId=111}
                         ↑ เดิม!

Jaeger เห็น:
  Trace abc:
    span 111 (Service A)
      └── span 222 (Service B)  ← เชื่อมกัน!
```

---

### ชิ้นที่ 4: `propagation.Baggage{}`

#### คืออะไร?

**Baggage** = implementation ของ **W3C Baggage Standard**

มันรู้วิธี:
- เขียน key-value metadata ลงใน `baggage` header
- อ่าน `baggage` header กลับมาใส่ใน context

```
Baggage{} รับผิดชอบ 1 header:

  baggage: userId=user_789,tenantId=tenant_abc,region=asia
```

#### Baggage คืออะไรในแง่ข้อมูล?

```
Baggage = key-value metadata ที่ "ติดมากับ request"
          และส่งต่อทุก service ในสาย chain โดยอัตโนมัติ

ตัวอย่าง:
  Request เริ่มที่ API Gateway:
    baggage: userId=user_789, tenantId=tenant_abc

  API Gateway → Order Service:
    baggage header ถูก forward ไปด้วย (อัตโนมัติถ้า inject)

  Order Service → DB Service:
    baggage header ยังคงอยู่

  ทุก service สามารถ GET ค่า baggage ออกมาใช้ได้:
    bag := baggage.FromContext(ctx)
    userId := bag.Member("userId").Value()  // "user_789"
```

---

## คำถามสำคัญ: Baggage vs Span Attribute ต่างกันยังไง ใช้อะไรตอนไหน?

### เปรียบเทียบโดยตรง

```
┌──────────────────────┬─────────────────────────┬──────────────────────────┐
│ Feature              │ Baggage                 │ Span Attribute           │
├──────────────────────┼─────────────────────────┼──────────────────────────┤
│ เก็บที่ไหน          │ HTTP Header             │ ใน Span (Jaeger เท่านั้น)│
│ ส่งข้าม service     │ ✅ ส่งไปทุก service     │ ❌ ไม่ส่ง               │
│ service อื่นเห็นได้ │ ✅ ทุกตัวในสาย          │ ❌ เฉพาะ service นั้น   │
│ เห็นใน Jaeger       │ ❌ (ต้อง set เอง)       │ ✅ เสมอ                  │
│ User เห็นได้         │ ⚠️ อยู่ใน Header       │ ✅ ปลอดภัยกว่า          │
│ ขนาด                │ จำกัด (~8KB header)     │ ใหญ่ได้มากกว่า          │
│ ใช้ใน business logic│ ✅ ดึงมาใช้ใน code ได้  │ ❌ ดูได้แค่ใน Jaeger    │
└──────────────────────┴─────────────────────────┴──────────────────────────┘
```

### ใช้ Baggage เมื่อ: "ต้องการให้ service อื่นรู้และใช้ค่านี้ใน code"

```go
// ตัวอย่าง: Multi-tenant system
// API Gateway รู้ tenantId จาก JWT token
// ต้องการให้ทุก service กรอง data ตาม tenantId

// API Gateway — ใส่ baggage:
func authMiddleware(c *fiber.Ctx) error {
    claims := parseJWT(c.Get("Authorization"))

    member, _ := baggage.NewMember("tenantId", claims.TenantID)
    bag, _ := baggage.New(member)
    ctx := baggage.ContextWithBaggage(c.UserContext(), bag)
    c.SetUserContext(ctx)
    return c.Next()
}

// Order Service — ดึง baggage มาใช้ใน code:
func (r *repo) GetOrders(ctx context.Context) ([]Order, error) {
    bag := baggage.FromContext(ctx)
    tenantId := bag.Member("tenantId").Value()  // "tenant_abc"

    // ใช้ tenantId กรอง — ไม่ต้องส่งผ่าน function parameter!
    return r.db.Where("tenant_id = ?", tenantId).Find(&[]Order{})
}

// DB Service ก็ได้รับ tenantId เดิมผ่าน baggage โดยอัตโนมัติ
```

### ใช้ Span Attribute เมื่อ: "ต้องการ debug / monitor ใน Jaeger"

```go
// ตัวอย่าง: บันทึกข้อมูลสำหรับ debug
func (r *repo) CreateOrder(ctx context.Context, req CreateOrderReq) (*Order, error) {
    ctx, span := tracer.Start(ctx, "CreateOrder")
    defer span.End()

    // Span Attribute — เก็บไว้ดูใน Jaeger ไม่ส่งข้าม service
    span.SetAttributes(
        attribute.String("order.id", req.ID),
        attribute.Float64("order.amount", req.Amount),
        attribute.Int("order.items_count", len(req.Items)),
        attribute.String("db.statement", "INSERT INTO orders..."),
    )

    // ดู Jaeger → เห็นทุก attribute นี้ใน span
    // แต่ service อื่นไม่รู้ว่า amount เท่าไหร่ (ไม่จำเป็นต้องรู้)
}
```

### สรุปง่ายๆ

```
ถามตัวเองว่า: "service อื่นต้องรู้ค่านี้เพื่อทำงานต่อไหม?"

ใช่  → Baggage (ส่งข้าม service ผ่าน header)
ไม่  → Span Attribute (เก็บไว้ดู Jaeger อย่างเดียว)

ตัวอย่าง:
  tenantId  → Baggage    ← ทุก service ต้องรู้เพื่อ filter data
  userId    → Baggage    ← ต้องรู้เพื่อ audit log
  amount    → Attribute  ← แค่ debug ดูว่า order มีกี่บาท
  sql query → Attribute  ← แค่ debug ดู query
  error msg → Attribute  ← แค่ debug ดู error
```

---

## คำถาม: Baggage ต้องใช้ตลอดไหม หรือใช้ Span เอา หรือไม่ต้องก็ได้?

### คำตอบ: Baggage เป็น Optional ทั้งหมด

```
propagation.Baggage{} ใน SetTextMapPropagator:
  → ทำให้ระบบ "รองรับ" baggage ได้
  → แต่ baggage จะไม่มีอะไรถ้าเราไม่ set มันเอง

3 กรณี:

กรณีที่ 1: ไม่ใส่ Baggage{} เลย
  otel.SetTextMapPropagator(propagation.TraceContext{})
  → trace linking ยังทำงานปกติ
  → แค่ไม่รองรับ baggage header
  → ถ้า service อื่นส่ง baggage มา → ถูกละเลย

กรณีที่ 2: ใส่ Baggage{} แต่ไม่เคย set baggage ใน code
  otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
      propagation.TraceContext{},
      propagation.Baggage{},
  ))
  // ไม่มี code ไหน baggage.ContextWithBaggage(...) เลย
  → header ที่ส่งออกไป: มีแค่ traceparent, ไม่มี baggage
  → ระบบทำงานปกติ แค่ไม่มี metadata วิ่งข้าม service

กรณีที่ 3: ใส่ Baggage{} และ set baggage ใน code
  → header ที่ส่งออกไป: มีทั้ง traceparent และ baggage
  → ทุก service รับและส่งต่อ baggage

→ "ใส่ Baggage{} ไว้ก่อน" เป็น best practice
  เผื่อวันหน้าต้องการใช้ ไม่ต้องแก้ config
```

### Span ทำเองได้ไหม? ไม่ต้องมี Baggage?

```
Span ทำให้เองไม่ได้ในสิ่งที่ Baggage ทำ:

Span Attribute:
  ✅ เก็บข้อมูลใน Jaeger
  ❌ service B ไม่รู้ค่า attribute ของ service A
  ❌ ไม่มีวิธีส่ง attribute ข้าม service

ตัวอย่างที่ทำไม่ได้ด้วย Span เพียงอย่างเดียว:

  // Service A set attribute:
  span.SetAttributes(attribute.String("tenant_id", "tenant_abc"))

  // Service B จะอ่านค่านี้จาก span ของ A ได้ไหม?
  // ❌ ไม่ได้! Span ของ A ส่งไป Jaeger ไม่ได้ผ่าน Service B

  // ต้อง Baggage:
  // Service A ใส่ baggage → Service B extract ออกมาได้เลย
```

---

## ภาพรวมทั้งหมด — ทุกชิ้นเชื่อมกันยังไง

### โค้ด 4 บรรทัดนี้ทำให้เกิดอะไร

```go
otel.SetTextMapPropagator(
    propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ),
)
```

```
สิ่งที่เกิดขึ้นทันทีที่รันโค้ดนี้:

OTEL Global Registry:
  TextMapPropagator ← CompositeTextMapPropagator{
      [0]: TraceContext{}   รู้จัก header "traceparent", "tracestate"
      [1]: Baggage{}        รู้จัก header "baggage"
  }

ผลกระทบ:
  otelfiber.Middleware()    → ใช้ global propagator นี้ตอน Extract
  otelgrpc interceptor      → ใช้ global propagator นี้ตอน Inject/Extract
  manual Inject ที่เราเขียน → ใช้ global propagator นี้
```

### Flow แบบ End-to-End เมื่อมีโค้ดนี้

```
─────────────────────────────────────────────────────────────────────────
Scenario: Browser → Central Control API → Country gRPC Service
─────────────────────────────────────────────────────────────────────────

1. Browser ส่ง Request (ไม่มี header พิเศษ)
   POST /v1/companies HTTP/1.1
   Authorization: Bearer eyJ...

   ─────────────────────────────────────────────────────────────────────
2. otelfiber.Middleware() รับ:
   carrier.Get("traceparent") → ""   ← ไม่มี
   carrier.Get("baggage")     → ""   ← ไม่มี

   → TraceContext{}.Extract → ไม่มีอะไร
   → Baggage{}.Extract      → ไม่มีอะไร
   → ctx ยังว่าง (ไม่มี parent)

   otelfiber สร้าง root span:
     traceId  = "4bf92f35..." (random)
     spanId   = "aaa001"
     parentId = null

   ctx ตอนนี้มี: span{traceId=4bf92f35, spanId=aaa001}

   ─────────────────────────────────────────────────────────────────────
3. authMiddleware (ถ้ามี) ใส่ baggage:
   userId  := claims.UserID   → "user_789"
   tenantId:= claims.TenantID → "tenant_abc"

   member1, _ := baggage.NewMember("userId",   "user_789")
   member2, _ := baggage.NewMember("tenantId", "tenant_abc")
   bag, _ := baggage.New(member1, member2)
   ctx = baggage.ContextWithBaggage(ctx, bag)

   ctx ตอนนี้มี:
     span{traceId=4bf92f35, spanId=aaa001}
     baggage{userId=user_789, tenantId=tenant_abc}

   ─────────────────────────────────────────────────────────────────────
4. companyRepository.GetCountry(ctx) เรียก gRPC:

   otelgrpc.NewClientHandler() ทำ Inject:
   propagator := otel.GetTextMapPropagator()  ← global ที่เรา Set
   propagator.Inject(ctx, grpcMetadataCarrier)

   TraceContext{}.Inject:
     → metadata["traceparent"] = "00-4bf92f35...-aaa001-01"

   Baggage{}.Inject:
     → metadata["baggage"] = "userId=user_789,tenantId=tenant_abc"

   gRPC request ที่ออกไป:
     traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-aaa001-01
     baggage: userId=user_789,tenantId=tenant_abc

   ─────────────────────────────────────────────────────────────────────
5. Country gRPC Service รับ request:

   otelgrpc.NewServerHandler() ทำ Extract:
   propagator := otel.GetTextMapPropagator()  ← global ของ service นั้น
   ctx = propagator.Extract(ctx, grpcMetadataCarrier)

   TraceContext{}.Extract:
     → อ่าน traceparent → SpanContext{traceId=4bf92f35, parentId=aaa001}
     → ใส่ใน ctx

   Baggage{}.Extract:
     → อ่าน baggage → {userId=user_789, tenantId=tenant_abc}
     → ใส่ใน ctx

   สร้าง span:
     traceId  = "4bf92f35..."  ← เดิม!
     spanId   = "bbb002"       ← ใหม่
     parentId = "aaa001"       ← span ของ Central Control

   ─────────────────────────────────────────────────────────────────────
6. Country Service ดึง baggage ใน code:
   bag := baggage.FromContext(ctx)
   tenantId := bag.Member("tenantId").Value()  // "tenant_abc"
   userId   := bag.Member("userId").Value()    // "user_789"

   กรอง data ตาม tenantId ได้เลย โดยไม่ต้องส่งใน gRPC request body!

   ─────────────────────────────────────────────────────────────────────
7. Jaeger เห็น:
   Trace: 4bf92f3577b34da6a3ce929d0e0e4736

   span aaa001 (central-control)  [parentId: null]   100ms
     └── span bbb002 (country-service) [parentId: aaa001]  40ms

   ← เชื่อมกันสมบูรณ์!
```

---

## ตาราง: แต่ละชิ้นทำอะไร สรุปชัดๆ

```
┌────────────────────────────────────┬──────────────────────────────────────────┐
│  โค้ด                              │  ทำอะไร                                │
├────────────────────────────────────┼──────────────────────────────────────────┤
│  otel.SetTextMapPropagator(...)    │  บอก OTEL global ว่า "ใช้ตัวนี้         │
│                                    │  สำหรับ inject/extract header"           │
│                                    │  ← library ทั้งหมดดึงไปใช้เอง           │
├────────────────────────────────────┼──────────────────────────────────────────┤
│  NewCompositeTextMapPropagator(…)  │  รวม propagator หลายตัวให้ทำงาน          │
│                                    │  พร้อมกันในคำสั่งเดียว                  │
├────────────────────────────────────┼──────────────────────────────────────────┤
│  propagation.TraceContext{}        │  จัดการ traceparent header              │
│                                    │  เชื่อม trace ข้าม service              │
│                                    │  ← ขาดไม่ได้ ต้องมีเสมอ               │
├────────────────────────────────────┼──────────────────────────────────────────┤
│  propagation.Baggage{}             │  จัดการ baggage header                  │
│                                    │  ส่ง metadata ข้าม service              │
│                                    │  ← optional แต่ควรมีไว้เผื่อ            │
└────────────────────────────────────┴──────────────────────────────────────────┘
```

---

## คำถาม: ถ้าลบ Baggage{} ออก จะเกิดอะไร?

```go
// ลบ Baggage{} ออก:
otel.SetTextMapPropagator(propagation.TraceContext{})

สิ่งที่เกิด:
  ✅ trace linking ยังทำงาน (traceparent ยังส่ง)
  ❌ baggage header ที่ส่งเข้ามา → ถูกละเลยทั้งหมด
  ❌ baggage ที่ set ใน ctx → ไม่ถูก inject ลง header
  → ถ้าไม่ใช้ baggage เลยในโปรเจกต์ → ไม่มีผลอะไร
  → ถ้าใช้ baggage → ข้อมูลหาย!
```

## คำถาม: ถ้าลบ TraceContext{} ออก จะเกิดอะไร?

```go
// ลบ TraceContext{} ออก:
otel.SetTextMapPropagator(propagation.Baggage{})

สิ่งที่เกิด:
  ❌ traceparent header ไม่ถูกอ่าน/เขียน
  ❌ trace chain ขาดทุก service hop
  ❌ Jaeger จะเห็นแต่ trace แยกกัน ไม่เชื่อมกัน
  ✅ baggage ส่งได้ แต่ไม่มีประโยชน์ถ้า trace ขาด
  → อย่าทำเด็ดขาด!
```

---

## สรุปสั้น — จำแค่นี้พอ

```
โค้ดนี้:
  otel.SetTextMapPropagator(
      propagation.NewCompositeTextMapPropagator(
          propagation.TraceContext{},  // ← เชื่อม trace ข้าม service
          propagation.Baggage{},       // ← ส่ง metadata ข้าม service
      ),
  )

เปรียบเหมือน:
  ลงทะเบียนกับ OTEL global ว่า:
  "เวลา inject header → เขียน traceparent และ baggage"
  "เวลา extract header → อ่าน traceparent และ baggage"

library อย่าง otelfiber, otelgrpc จะใช้ global นี้อัตโนมัติ
เราไม่ต้องบอกทีละ library

TraceContext → บังคับต้องมี (ไม่มี = trace ไม่เชื่อมกัน)
Baggage     → optional (ใส่ไว้เผื่อ, ไม่มีผลถ้าไม่ใช้)
Baggage ≠ Span Attribute (Baggage ส่งข้าม service ได้, Attribute ไม่ได้)
```
