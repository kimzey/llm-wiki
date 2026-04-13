# OpenTelemetry — Deep Dive ฉบับ Part 2

> เจาะลึก TextMapPropagator, TextMapCarrier, การไหลของ Spans และ Jaeger Assembly

---

## สารบัญ

1. [TextMapCarrier — "กล่อง" ที่บรรจุ Context](#1-textmapcarrier--กล่องที่บรรจุ-context)
2. [TextMapPropagator — "ตรรกะ" ที่อ่านเขียน Context](#2-textmappropagator--ตรรกะที่อ่านเขียน-context)
3. [ทั้งสองทำงานร่วมกันยังไง?](#3-ทั้งสองทำงานร่วมกันยังไง)
4. [ใช้ตรงไหน พร้อมกันไหม แยกกันไหม?](#4-ใช้ตรงไหน-พร้อมกันไหมแยกกันไหม)
5. [Span ถูกส่งยังไง — ไม่ได้รอครบแล้วส่งทีเดียว!](#5-span-ถูกส่งยังไง--ไม่ได้รอครบแล้วส่งทีเดียว)
6. [Jaeger รวม Spans ยังไง — Assembly ที่ปลายทาง](#6-jaeger-รวม-spans-ยังไง--assembly-ที่ปลายทาง)
7. [Timeline จริง: ตั้งแต่ Request จนเห็นใน Jaeger](#7-timeline-จริง-ตั้งแต่-request-จนเห็นใน-jaeger)
8. [BatchSpanProcessor — กลไกการ Buffer ลึกๆ](#8-batchspanprocessor--กลไกการ-buffer-ลึกๆ)
9. [ตัวอย่างข้อมูลจริงทุกขั้นตอน](#9-ตัวอย่างข้อมูลจริงทุกขั้นตอน)

---

## 1. TextMapCarrier — "กล่อง" ที่บรรจุ Context

### คืออะไร?

**TextMapCarrier** คือ **interface** ที่เป็น abstraction ของ "สิ่งที่เก็บ key-value string"

มันคือ "กล่อง" ที่ใช้ขนส่ง trace context ไปกับ request
ตัวกล่องเองไม่รู้ว่าข้างในคืออะไร — แค่รู้ว่า Get/Set key-value ได้

```go
// TextMapCarrier interface ใน OTEL:
type TextMapCarrier interface {
    Get(key string) string        // อ่านค่าตาม key
    Set(key string, value string) // เขียนค่าลง key
    Keys() []string               // รายชื่อ key ทั้งหมดที่มี
}
```

### แต่ละ method ทำอะไร?

```
Get(key string) string
  ─────────────────────
  หน้าที่: อ่านค่าตาม key ที่ระบุ
  ใช้ตอน: Extract — Propagator ถามว่า "ใน header มี traceparent ไหม?"
  ถ้าไม่มี key นั้น: return ""

  ตัวอย่าง:
    carrier.Get("traceparent")
    → "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"

    carrier.Get("baggage")
    → "userId=789,tenantId=abc"

    carrier.Get("x-not-exist")
    → ""

Set(key string, value string)
  ────────────────────────────
  หน้าที่: เขียนค่าลง key
  ใช้ตอน: Inject — Propagator บอกว่า "เขียน traceparent นี้ลง header"
  Note: case-insensitive สำหรับ HTTP (ตาม HTTP spec)

  ตัวอย่าง:
    carrier.Set("traceparent", "00-4bf92f35...-00f067aa...-01")
    carrier.Set("baggage", "userId=789,tenantId=abc")

Keys() []string
  ─────────────
  หน้าที่: คืนรายชื่อ key ทั้งหมดที่ carrier มีอยู่
  ใช้ตอน: Extract — Propagator วน loop ดูว่ามี key ไหนที่ตัวเองสนใจบ้าง
  ใช้น้อยกว่า Get/Set มาก (บาง Propagator ไม่ใช้เลย)

  ตัวอย่าง:
    carrier.Keys()
    → ["content-type", "authorization", "traceparent", "baggage"]
```

### Implementation ของ TextMapCarrier มีหลายแบบ

```go
// ─── 1. HeaderCarrier — ครอบ http.Header ───
// ใช้กับ HTTP request/response
type HeaderCarrier http.Header

func (hc HeaderCarrier) Get(key string) string {
    return http.Header(hc).Get(key)  // case-insensitive
}
func (hc HeaderCarrier) Set(key string, value string) {
    http.Header(hc).Set(key, value)
}
func (hc HeaderCarrier) Keys() []string {
    keys := make([]string, 0, len(hc))
    for k := range hc {
        keys = append(keys, k)
    }
    return keys
}

// ─── 2. MapCarrier — ครอบ map[string]string ───
// ใช้กับ Kafka message headers, gRPC metadata, หรือ test
type MapCarrier map[string]string

func (mc MapCarrier) Get(key string) string     { return mc[key] }
func (mc MapCarrier) Set(key, value string)     { mc[key] = value }
func (mc MapCarrier) Keys() []string {
    keys := make([]string, 0, len(mc))
    for k := range mc { keys = append(keys, k) }
    return keys
}

// ─── ทั้งสองมีอยู่ใน OTEL SDK แล้ว ───
// import "go.opentelemetry.io/otel/propagation"
// propagation.HeaderCarrier(req.Header)
// propagation.MapCarrier(myMap)
```

### ตัวอย่างข้อมูลใน Carrier แต่ละแบบ

```
HTTP Request Headers (HeaderCarrier):
  ┌─────────────────────────────────────────────────────────────────┐
  │ POST /v1/orders HTTP/1.1                                        │
  │ Host: order-service:8080                                        │
  │ Content-Type: application/json                                  │
  │ Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9... │
  │ traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa-01   │ ← trace context
  │ baggage: userId=user_789,tenantId=tenant_abc                    │ ← baggage
  │                                                                 │
  │ {"amount": 299.99}                                              │
  └─────────────────────────────────────────────────────────────────┘
  carrier.Get("traceparent") → "00-4bf92f35...-00f067aa...-01"
  carrier.Get("baggage")     → "userId=user_789,tenantId=tenant_abc"
  carrier.Get("content-type")→ "application/json"

Kafka Message Headers (MapCarrier):
  ┌─────────────────────────────────────────────────────────────────┐
  │ Topic: order.created                                            │
  │ Headers:                                                        │
  │   "traceparent" → "00-4bf92f35...-a3ce929d...-01"              │
  │   "baggage"     → "userId=user_789,tenantId=tenant_abc"        │
  │   "content-type"→ "application/json"                           │
  │ Value: {"orderId": "ord_abc", "amount": 299.99}                 │
  └─────────────────────────────────────────────────────────────────┘
```

---

## 2. TextMapPropagator — "ตรรกะ" ที่อ่านเขียน Context

### คืออะไร?

**TextMapPropagator** คือ **interface** ที่มี logic สำหรับ:
- **Inject**: เอา trace context จาก `ctx` → เขียนลงใน carrier
- **Extract**: อ่าน trace context จาก carrier → ใส่กลับเข้า `ctx`

มันคือ "คนที่รู้ format" — รู้ว่า traceparent หน้าตาเป็นยังไง, parse ยังไง, validate ยังไง

```go
// TextMapPropagator interface ใน OTEL:
type TextMapPropagator interface {
    Inject(ctx context.Context, carrier TextMapCarrier)
    Extract(ctx context.Context, carrier TextMapCarrier) context.Context
    Fields() []string
}
```

### แต่ละ method ทำอะไร?

```
Inject(ctx context.Context, carrier TextMapCarrier)
  ──────────────────────────────────────────────────
  หน้าที่: ดึง SpanContext จาก ctx แล้วเขียนลง carrier
  ใช้ตอน: "กำลังจะส่ง request ออกไป" → inject ก่อนส่ง
  Input:  ctx ที่มี active span, carrier ที่จะเขียนลง
  Output: ไม่มี (เขียนลง carrier โดยตรง — side effect)

  กระบวนการภายใน:
    1. ดึง SpanContext จาก ctx
    2. ตรวจว่า SpanContext valid ไหม (มี traceId, spanId)
    3. แปลงเป็น string format: "00-{traceId}-{spanId}-{flags}"
    4. เรียก carrier.Set("traceparent", formattedString)
    5. ถ้ามี tracestate → carrier.Set("tracestate", ...)
    6. ถ้ามี baggage → carrier.Set("baggage", ...)

Extract(ctx context.Context, carrier TextMapCarrier) context.Context
  ──────────────────────────────────────────────────────────────────
  หน้าที่: อ่านจาก carrier แล้วสร้าง SpanContext ใส่กลับใน ctx
  ใช้ตอน: "รับ request เข้ามา" → extract ก่อน handle
  Input:  ctx เปล่า (หรือมี context อื่น), carrier จาก incoming request
  Output: ctx ใหม่ที่มี SpanContext จาก carrier ฝังอยู่

  กระบวนการภายใน:
    1. เรียก carrier.Get("traceparent")
    2. ถ้าไม่มี → return ctx เดิม (จะกลายเป็น root span ตอน Start)
    3. parse string: split ด้วย "-" → version, traceId, parentSpanId, flags
    4. validate แต่ละส่วน (ต้องไม่ all-zeros, length ถูกต้อง)
    5. สร้าง SpanContext{TraceID, SpanID, TraceFlags, ...}
    6. return ctx ใหม่ที่มี SpanContext นี้ฝังอยู่

Fields() []string
  ──────────────────
  หน้าที่: คืนชื่อ header keys ที่ propagator นี้ใช้
  ใช้ตอน: ตอนจะ expose CORS headers หรือ log ว่า propagate อะไรบ้าง

  ตัวอย่าง TraceContext propagator:
    → ["traceparent", "tracestate"]

  ตัวอย่าง Baggage propagator:
    → ["baggage"]

  ตัวอย่าง Composite (TraceContext + Baggage):
    → ["traceparent", "tracestate", "baggage"]
```

### Implementation ภายในของ TraceContext Propagator

```
Inject จริงๆ ทำแบบนี้:

  ได้รับ ctx ที่มี SpanContext:
    TraceID = [16]byte{0x4b, 0xf9, 0x2f, ...}
    SpanID  = [8]byte{0x00, 0xf0, 0x67, ...}
    Flags   = TraceFlags(0x01)  // sampled

  แปลงเป็น hex string:
    traceId = "4bf92f3577b34da6a3ce929d0e0e4736"
    spanId  = "00f067aa0ba902b7"
    flags   = "01"

  สร้าง traceparent:
    "00" + "-" + traceId + "-" + spanId + "-" + flags
    = "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"

  เรียก:
    carrier.Set("traceparent", "00-4bf92f...01")

Extract จริงๆ ทำแบบนี้:

  เรียก:
    raw = carrier.Get("traceparent")
    → "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"

  parse:
    parts = split(raw, "-")  → ["00", "4bf92f...", "00f067...", "01"]
    version  = parts[0]  → "00"  (validate ≠ "ff")
    traceId  = parts[1]  → parse 32 hex → [16]byte
    spanId   = parts[2]  → parse 16 hex → [8]byte
    flags    = parts[3]  → parse 2 hex  → TraceFlags

  validate:
    traceId ≠ all zeros → ✅
    spanId  ≠ all zeros → ✅
    version = "00"      → ✅

  สร้าง SpanContext:
    SpanContext{
      TraceID:    [16]byte{0x4b, 0xf9...},
      SpanID:     [8]byte{0x00, 0xf0...},
      TraceFlags: TraceFlags(0x01),
      Remote:     true,  // ← บอกว่า context นี้มาจากภายนอก
    }

  ฝังใน ctx:
    return context.WithValue(ctx, spanContextKey{}, spanCtx)
```

---

## 3. ทั้งสองทำงานร่วมกันยังไง?

### สรุปความสัมพันธ์

```
TextMapCarrier  = "กล่อง" (สิ่งที่เก็บ key-value)
TextMapPropagator = "คนขนของ" (รู้ว่าต้องเอาอะไรใส่กล่อง ใส่ยังไง อ่านยังไง)

ขาดอันใดอันหนึ่งไม่ได้:
  - มีแต่ Carrier: มีกล่องแต่ไม่รู้จะใส่อะไร
  - มีแต่ Propagator: รู้จะใส่อะไรแต่ไม่มีกล่อง
  - มีทั้งคู่: ✅ ทำงานได้

Propagator ใช้ Carrier เป็น "เครื่องมือ"
Carrier ไม่รู้จัก Propagator
```

### Diagram ความสัมพันธ์

```
┌────────────────────────────────────────────────────────────────┐
│                      INJECT FLOW                               │
│                                                                │
│  ctx (มี SpanContext อยู่ข้างใน)                               │
│    │                                                           │
│    ▼                                                           │
│  Propagator.Inject(ctx, carrier)                               │
│    │                                                           │
│    ├─ ดึง SpanContext จาก ctx                                  │
│    ├─ แปลงเป็น string format                                  │
│    └─ เรียก carrier.Set("traceparent", "00-abc-def-01")       │
│              │                                                 │
│              ▼                                                 │
│         HTTP Header                                            │
│         "traceparent: 00-abc-def-01"  ← ออกไปกับ request      │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│                      EXTRACT FLOW                              │
│                                                                │
│  HTTP Header เข้ามา                                           │
│  "traceparent: 00-abc-def-01"                                  │
│    │                                                           │
│    ▼                                                           │
│  carrier = HeaderCarrier(request.Header)                       │
│    │                                                           │
│    ▼                                                           │
│  newCtx = Propagator.Extract(ctx, carrier)                     │
│    │                                                           │
│    ├─ เรียก carrier.Get("traceparent")                         │
│    ├─ parse → SpanContext{traceId=abc, parentId=def}           │
│    └─ ใส่ SpanContext กลับใน ctx ใหม่                         │
│              │                                                 │
│              ▼                                                 │
│         newCtx (มี SpanContext พร้อม traceId แล้ว)             │
│    เอา newCtx ไปทำ tracer.Start(newCtx, ...) ต่อ              │
└────────────────────────────────────────────────────────────────┘
```

---

## 4. ใช้ตรงไหน พร้อมกันไหม แยกกันไหม?

### คำตอบสั้น

```
ใช้พร้อมกันเสมอ แต่คนละสถานการณ์:
  Inject → ตอนเป็น "ผู้ส่ง" (outbound)
  Extract → ตอนเป็น "ผู้รับ" (inbound)

ทำแยก side กัน:
  Service A (caller):  ทำ Inject ก่อน request ออก
  Service B (callee):  ทำ Extract ตอน request เข้ามา
```

### Map ว่าใช้ตรงไหน ใครทำ

```
─────────────────────────────────────────────────────────────────
ฝั่ง                 Operation    ใครทำ
─────────────────────────────────────────────────────────────────
HTTP Server รับ      Extract      otelfiber.Middleware() อัตโนมัติ
HTTP Client ส่ง      Inject       ต้องทำเอง หรือใช้ otelhttp.Transport
gRPC Server รับ      Extract      otelgrpc.NewServerHandler() อัตโนมัติ
gRPC Client ส่ง      Inject       otelgrpc.NewClientHandler() อัตโนมัติ
Kafka Producer       Inject       otelsarama หรือทำเอง
Kafka Consumer       Extract      otelsarama หรือทำเอง
─────────────────────────────────────────────────────────────────
```

### โค้ดทุก Case

```go
// ═══════════════════════════════════════════════════════════════
// CASE 1: HTTP Server รับ request — Extract (otelfiber ทำให้)
// ═══════════════════════════════════════════════════════════════

// fiber_server.go บรรทัดนี้ทำ Extract อัตโนมัติ:
server.Use(otelfiber.Middleware())

// otelfiber ทำแบบนี้ภายใน (pseudocode):
func otelMiddleware(c *fiber.Ctx) error {
    // 1. ครอบ fiber request header เป็น carrier
    carrier := propagation.HeaderCarrier(c.Request().Header)

    // 2. extract trace context จาก carrier
    ctx := otel.GetTextMapPropagator().Extract(c.Context(), carrier)

    // 3. สร้าง span สำหรับ HTTP request นี้
    ctx, span := tracer.Start(ctx, c.Method()+" "+c.Path(),
        trace.WithSpanKind(trace.SpanKindServer),
    )
    defer span.End()

    // 4. วาง ctx ใน fiber context ให้ handler ใช้
    c.SetUserContext(ctx)
    return c.Next()
}

// ═══════════════════════════════════════════════════════════════
// CASE 2: HTTP Client ส่ง request — Inject (ทำเอง)
// ═══════════════════════════════════════════════════════════════

func (r *managementRepo) GetUser(ctx context.Context, id string) (*User, error) {
    req, _ := http.NewRequestWithContext(ctx, "GET", r.baseURL+"/users/"+id, nil)

    // Inject: เขียน traceparent ลง HTTP header
    //   Propagator ดึง SpanContext จาก ctx
    //   แปลงเป็น "00-traceId-spanId-01"
    //   เรียก header.Set("traceparent", ...)
    otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))

    // ตอนนี้ req.Header["Traceparent"] = "00-4bf92f35...-00f067aa...-01"
    resp, err := r.client.Do(req)
    // ...
}

// ─── ทำแบบ automatic ด้วย otelhttp.Transport ───
r.client = &http.Client{
    Transport: otelhttp.NewTransport(http.DefaultTransport),
    // ทุก request จาก client นี้จะ Inject อัตโนมัติ
}

// ═══════════════════════════════════════════════════════════════
// CASE 3: gRPC Client ส่ง — Inject อัตโนมัติ
// ═══════════════════════════════════════════════════════════════

conn, _ := grpc.NewClient(addr,
    grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
    // ทุก gRPC call จะ Inject trace context ใน metadata อัตโนมัติ
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)

// gRPC metadata ที่ถูก inject:
// grpc-trace-bin: <binary encoded trace context>
// (หรือ traceparent ถ้าใช้ W3C propagator)

// ═══════════════════════════════════════════════════════════════
// CASE 4: Kafka Producer — Inject ด้วยตัวเอง
// ═══════════════════════════════════════════════════════════════

func (p *producer) PublishOrderCreated(ctx context.Context, order Order) error {
    // สร้าง map สำหรับเก็บ headers
    carrier := propagation.MapCarrier{}

    // Inject: เขียน traceparent ลง map
    otel.GetTextMapPropagator().Inject(ctx, carrier)
    // carrier = {"traceparent": "00-abc...", "baggage": "userId=..."}

    // แปลง map เป็น Kafka headers
    var headers []sarama.RecordHeader
    for k, v := range carrier {
        headers = append(headers, sarama.RecordHeader{
            Key:   []byte(k),
            Value: []byte(v),
        })
    }

    msg := &sarama.ProducerMessage{
        Topic:   "order.created",
        Value:   sarama.ByteEncoder(orderJSON),
        Headers: headers,  // ← trace context อยู่ที่นี่
    }
    _, _, err := p.producer.SendMessage(msg)
    return err
}

// ═══════════════════════════════════════════════════════════════
// CASE 5: Kafka Consumer — Extract ด้วยตัวเอง
// ═══════════════════════════════════════════════════════════════

func (w *worker) ProcessMessage(msg *sarama.ConsumerMessage) {
    // แปลง Kafka headers เป็น map
    carrier := propagation.MapCarrier{}
    for _, h := range msg.Headers {
        carrier[string(h.Key)] = string(h.Value)
    }
    // carrier = {"traceparent": "00-abc...", "baggage": "userId=..."}

    // Extract: อ่าน traceparent จาก map → ใส่ใน ctx
    ctx := otel.GetTextMapPropagator().Extract(context.Background(), carrier)
    // ctx ตอนนี้มี SpanContext ที่มี traceId จาก producer

    // สร้าง span ต่อจาก producer's span
    ctx, span := tracer.Start(ctx, "ProcessOrderCreated",
        trace.WithSpanKind(trace.SpanKindConsumer),
    )
    defer span.End()

    w.useCase.ProcessOrder(ctx, parseOrder(msg.Value))
}

// ═══════════════════════════════════════════════════════════════
// CASE 6: Test — ใช้ MapCarrier simulate headers
// ═══════════════════════════════════════════════════════════════

func TestPropagation(t *testing.T) {
    // simulate: Service A inject ลงไป
    ctx, span := tracer.Start(context.Background(), "parent")

    carrier := propagation.MapCarrier{}
    otel.GetTextMapPropagator().Inject(ctx, carrier)

    span.End()

    // ตรวจว่า carrier มีอะไร
    assert.NotEmpty(t, carrier["traceparent"])
    // carrier["traceparent"] = "00-{traceId}-{spanId}-01"

    // simulate: Service B extract ออกมา
    newCtx := otel.GetTextMapPropagator().Extract(context.Background(), carrier)
    newSpanCtx := trace.SpanContextFromContext(newCtx)

    // ต้องได้ traceId เดิม
    assert.Equal(t, span.SpanContext().TraceID(), newSpanCtx.TraceID())
}
```

### สรุปว่าใช้ตรงไหน

```
┌────────────────────┬──────────────────┬──────────────────────┐
│  สถานการณ์         │  Operation       │  ใครทำ              │
├────────────────────┼──────────────────┼──────────────────────┤
│ รับ HTTP request   │ Extract          │ otelfiber (auto)     │
│ ส่ง HTTP request   │ Inject           │ otelhttp / ทำเอง    │
│ รับ gRPC request   │ Extract          │ otelgrpc (auto)      │
│ ส่ง gRPC request   │ Inject           │ otelgrpc (auto)      │
│ Kafka publish      │ Inject           │ otelsarama / ทำเอง  │
│ Kafka consume      │ Extract          │ otelsarama / ทำเอง  │
│ Unit test simulate │ Inject + Extract │ ทำเอง (MapCarrier)  │
└────────────────────┴──────────────────┴──────────────────────┘

กฎ: ผู้ "ส่ง" ทำ Inject, ผู้ "รับ" ทำ Extract
    ไม่มีทางที่ 1 process จะทำ Inject + Extract พร้อมกัน
    (ยกเว้น test)
```

---

## 5. Span ถูกส่งยังไง — ไม่ได้รอครบแล้วส่งทีเดียว!

### ความเข้าใจผิดที่พบบ่อย

```
❌ ความเชื่อผิด:
  "SDK รอให้ spans ทั้งหมดของ trace ครบก่อน
   แล้วค่อยรวมแล้วส่งไป Jaeger ทีเดียว"

✅ ความจริง:
  "แต่ละ Service ส่ง spans ของตัวเองไป Jaeger อิสระจากกัน
   ทันทีที่ span.End() ถูกเรียก span จะถูก queue ไว้ใน BatchProcessor
   Jaeger รับ spans จากหลาย service แยกกัน
   แล้วค่อย group เองตอนแสดงผล โดยใช้ traceId เป็นตัวเชื่อม"
```

### Timeline จริงของ Span ตั้งแต่ถูกสร้างถึง Jaeger

```
T=0ms   tracer.Start(ctx, "HTTP POST /orders")
           → สร้าง Span object ใน memory
           → ยัง "in-flight" อยู่ ยังไม่ส่งไปไหน

T=50ms  tracer.Start(ctx, "DB INSERT orders")
           → สร้าง child Span ใน memory

T=120ms span.End()  ← DB span จบ
           → คำนวณ duration = 70ms
           → ส่ง span นี้ไปหา BatchSpanProcessor
           → BatchSpanProcessor เพิ่มลง queue/buffer

T=150ms span.End()  ← HTTP span จบ
           → คำนวณ duration = 150ms
           → ส่งไป BatchSpanProcessor อีกตัว
           → BatchSpanProcessor เพิ่มลง queue/buffer

T=5000ms BatchSpanProcessor flush (ทุก 5 วิ):
           → รวม spans ที่ค้างอยู่ใน buffer เป็น batch
           → ส่งผ่าน OTLP/gRPC ไป OTEL Collector
           → Collector forward ไป Jaeger

(หรือ flush ทันทีถ้า buffer เต็ย 512 spans)
```

### BatchSpanProcessor — buffer อยู่ใน process

```
┌──────────────────────────────────────────────────────────────┐
│                    Go Application Process                    │
│                                                              │
│  span.End() ──▶ SpanProcessor.OnEnd(span)                   │
│                      │                                       │
│                      ▼                                       │
│              ┌───────────────────┐                           │
│              │  Internal Buffer  │                           │
│              │  (queue/channel)  │                           │
│              │                   │                           │
│              │  [span1] [span2]  │  ← spans นั่งรออยู่      │
│              │  [span3] [span4]  │                           │
│              │  [span5]          │                           │
│              └──────────┬────────┘                           │
│                         │                                    │
│              trigger flush เมื่อ:                           │
│              ① ครบ 5 วินาที (timeout)                       │
│              ② buffer มี 512 spans (batch size)             │
│              ③ tp.Shutdown() ถูกเรียก                       │
│                         │                                    │
│                         ▼                                    │
│              ส่งเป็น batch ผ่าน gRPC ──▶ OTEL Collector     │
└──────────────────────────────────────────────────────────────┘
```

### หลาย Service ส่ง Spans แยกกัน คนละเวลา

```
Timeline จริงเมื่อ Request ผ่าน 3 Service:

Service A (api-gateway)    Service B (order-service)   Service C (db-service)
─────────────────────────  ──────────────────────────  ───────────────────────
T=0    span.Start (root)
T=10   → เรียก Service B
                           T=10  span.Start
                           T=30  → query DB
                                                        T=30  span.Start
                                                        T=80  span.End ──▶ buffer
                           T=80  span.End ──▶ buffer
T=100  span.End ──▶ buffer

T=5000 flush ──▶ Collector  T=5000 flush ──▶ Collector  T=5000 flush ──▶ Collector
                ↓                          ↓                          ↓
              Jaeger        ←─── รับ batch ทั้ง 3 service แยกกัน ──→

Jaeger ดูทุก batch:
  - span จาก A: traceId=abc, spanId=111, parentId=null
  - span จาก B: traceId=abc, spanId=222, parentId=111
  - span จาก C: traceId=abc, spanId=333, parentId=222

Jaeger ตรวจ traceId → abc เหมือนกันทั้งหมด
  → group เข้า Trace "abc"
  → เรียงตาม parentId → สร้าง tree

ผลลัพธ์ใน UI:
  abc
  └── 111 (A) 100ms
       └── 222 (B) 70ms
            └── 333 (C) 50ms
```

---

## 6. Jaeger รวม Spans ยังไง — Assembly ที่ปลายทาง

### Jaeger Storage Model

```
Jaeger ไม่ได้เก็บข้อมูลเป็น "trace object"
แต่เก็บเป็น "spans" แยกกัน index ด้วย traceId

Storage (conceptually):
  Table: spans
  ┌─────────────┬──────────┬──────────┬────────────────┬──────────┐
  │  traceId    │  spanId  │ parentId │  operationName │ duration │
  ├─────────────┼──────────┼──────────┼────────────────┼──────────┤
  │ 4bf92f35... │ 111      │ null     │ HTTP POST /ord │ 100ms    │
  │ 4bf92f35... │ 222      │ 111      │ DB INSERT      │ 70ms     │
  │ 4bf92f35... │ 333      │ 222      │ SELECT users   │ 50ms     │
  │ deadbeef... │ 444      │ null     │ HTTP GET /user │ 30ms     │
  └─────────────┴──────────┴──────────┴────────────────┴──────────┘

ตอน user ค้นหา Trace "4bf92f35...":
  1. Jaeger query: WHERE traceId = "4bf92f35..."
  2. ได้ spans: [111, 222, 333]
  3. sort ด้วย startTime
  4. สร้าง tree ด้วย parentId
  5. แสดงผล
```

### Jaeger ทำ Late Assembly — รวมทีหลัง

```
"Late Assembly" = รับ spans ที่วิ่งเข้ามาเรื่อยๆ เก็บใน storage
                  ตอน query ค่อยดึงมา assemble เป็น trace

ข้อดี:
  ✅ ไม่ต้อง block รอ spans ทั้งหมดก่อน
  ✅ Service ที่ crash กลางคัน → ยัง query trace ที่ไม่สมบูรณ์ได้
  ✅ Scale ได้ง่าย (spans เข้ามาเรื่อยๆ แค่ insert)

ข้อเสีย:
  ❌ Tail-based sampling ยากกว่า (ต้องรอ trace สมบูรณ์)
  ❌ ถ้า span มาช้ามาก (network lag) อาจหาย
```

### กรณีพิเศษ: Span มาช้า

```
T=0s    Service A ส่ง span ไป Jaeger
T=1s    Service B ส่ง span ไป Jaeger
T=30s   Service C ส่ง span ไป Jaeger  ← มาช้ามาก (network issue)

User ค้นหา trace ที่ T=10s:
  → เห็น span A + B แต่ไม่เห็น C (ยังไม่ถึง Jaeger)

User ค้นหา trace ที่ T=35s:
  → เห็นครบทั้ง A + B + C

→ Jaeger แสดงผลตาม "สิ่งที่รับมาแล้ว ณ ตอนนั้น"
→ spans ไม่หายถาวร แค่มาช้า
```

---

## 7. Timeline จริง: ตั้งแต่ Request จนเห็นใน Jaeger

### ตัวอย่างสมบูรณ์ — POST /v1/companies

```
User กด Submit Form (T=0ms)
    │
    ▼
Browser ส่ง HTTP Request:
  POST /v1/companies HTTP/1.1
  (ไม่มี traceparent — Browser ยังไม่มี OTEL)

    │ T=1ms
    ▼
fiber otelfiber.Middleware()
  carrier = HeaderCarrier(request.Headers)
  carrier.Get("traceparent") → ""  (ไม่มี)
  → สร้าง root span ใหม่:
      traceId = "4bf92f3577b34da6a3ce929d0e0e4736"  (random UUID)
      spanId  = "aaaaaaaaaaaa0001"
      parentId = null
  → span ยัง "in-flight" ใน memory

    │ T=2ms
    ▼
companyHandler.Create(ctx)  ← ctx มี span "aaaaaa0001" อยู่
  useCase.Create(ctx, req)

    │ T=3ms
    ▼
companyUseCase.Create(ctx, req)
  repo.Insert(ctx, company)

    │ T=4ms
    ▼
companyRepository.Insert(ctx, company)
  ctx, span2 = tracer.Start(ctx, "Insert")
      traceId = "4bf92f35..."  ← เดิม (ดึงจาก ctx)
      spanId  = "bbbbbbbbbbbb0002"
      parentId = "aaaaaaaaaaaa0001"
  → span2 ยัง "in-flight" ใน memory

  db.WithContext(ctx).Create(&company)

    │ T=74ms
    ▼
  SQL "INSERT INTO companies..." เสร็จ

    │ T=74ms
    ▼
  span2.End()
  → duration = 70ms
  → ส่งไปหา BatchSpanProcessor
  → BatchSpanProcessor เพิ่ม span2 ลง buffer

    │ T=75ms
    ▼
companyRepository.Insert กลับมา

    │ T=100ms
    ▼
otelfiber middleware หลัง c.Next() เสร็จ
  span1.SetAttributes(http.StatusCode = 201)
  span1.SetStatus(codes.Ok, "")
  span1.End()
  → duration = 100ms
  → ส่งไปหา BatchSpanProcessor
  → BatchSpanProcessor เพิ่ม span1 ลง buffer

    │ (ผ่านไป 5 วิ)  T=5100ms
    ▼
BatchSpanProcessor Ticker ดัง:
  flush buffer: [span2 (DB), span1 (HTTP)]
  serialize เป็น OTLP Protobuf binary
  ส่งผ่าน gRPC ไป OTEL Collector ที่ localhost:4317

    │ T=5101ms
    ▼
OTEL Collector รับ batch:
  pipeline:
    ① filter processor: /health? → ไม่ใช่ → ผ่าน
    ② batch processor: รวมกับ spans อื่นๆ ที่รอ
    ③ otlp/jaeger exporter: ส่งต่อไป Jaeger

    │ T=5102ms
    ▼
Jaeger รับ spans:
  INSERT span: traceId=4bf92f35, spanId=bb0002, parentId=aa0001, dur=70ms
  INSERT span: traceId=4bf92f35, spanId=aa0001, parentId=null,   dur=100ms

    │ (user เปิด Jaeger UI)
    ▼
User ค้นหา:
  Service: sellsuki-central-control-backend-development
  Find Traces

Jaeger query:
  SELECT * FROM spans WHERE traceId = "4bf92f35..."
  → [span aa0001, span bb0002]
  → sort by startTime
  → build tree by parentId

Jaeger แสดงผล:
  Trace: 4bf92f3577b34da6a3ce929d0e0e4736  [100ms]

  [0ms──────────────────────────────────]100ms
  HTTP POST /v1/companies                   100ms (api)

    [4ms─────────────────────────]74ms
    Insert                                   70ms (repo→db)
```

---

## 8. BatchSpanProcessor — กลไกการ Buffer ลึกๆ

### โครงสร้างภายใน

```
BatchSpanProcessor มี:
  - exportQueue   chan ReadOnlySpan   → channel รับ spans
  - timer         *time.Ticker       → trigger flush ทุก N วิ
  - goroutine     background worker  → วน loop drain queue + flush

flow:
  span.End()
    → OnEnd(span) ถูกเรียก
    → ตรวจ sampling: sampled? → ใส่ลง exportQueue channel
    → ถ้า channel เต็ม → DROP span (!) ← ระวัง!

  background goroutine:
    loop:
      select:
        case span = <-exportQueue:
          batch = append(batch, span)
          if len(batch) >= maxBatchSize:
            flush(batch)
            batch = []

        case <-ticker.C:
          if len(batch) > 0:
            flush(batch)
            batch = []

        case <-stopCh:
          flush(batch)  ← flush ก่อนตาย
          return
```

### Configuration ที่ควรรู้

```go
import sdktrace "go.opentelemetry.io/otel/sdk/trace"

tp := trace.NewTracerProvider(
    trace.WithBatcher(exporter,
        // ─── Timing ───
        sdktrace.WithBatchTimeout(5 * time.Second),   // flush ทุก 5 วิ (default)
        sdktrace.WithExportTimeout(30 * time.Second), // timeout ของการ export

        // ─── Batch Size ───
        sdktrace.WithMaxExportBatchSize(512),  // ส่งทีละ 512 spans (default)
        sdktrace.WithMaxQueueSize(2048),        // buffer สูงสุด (default)
        // ถ้า queue เต็ม → spans ที่เข้ามาใหม่จะถูก DROP!
    ),
)

// ─── ตัวอย่างปรับสำหรับ high traffic ───
sdktrace.WithMaxQueueSize(10000),       // buffer ใหญ่ขึ้น
sdktrace.WithMaxExportBatchSize(1000),  // batch ใหญ่ขึ้น
sdktrace.WithBatchTimeout(2 * time.Second), // flush บ่อยขึ้น
```

### SimpleSpanProcessor vs BatchSpanProcessor

```
SimpleSpanProcessor:
  span.End() → ส่งทันที (synchronous)

  ข้อดี:
    ✅ เห็นใน Jaeger ทันที (development ดี)
    ✅ ไม่มี delay
  ข้อเสีย:
    ❌ ทุก span = 1 network call → ช้า
    ❌ ถ้า Collector ล่ม → span.End() block!
    ❌ ใช้ใน production ไม่ได้

BatchSpanProcessor:
  span.End() → ใส่ queue → flush เป็น batch ทุก 5 วิ

  ข้อดี:
    ✅ ไม่ block application
    ✅ ถ้า Collector ล่ม → spans queue อยู่ใน buffer (ไม่ block)
    ✅ Network efficient
  ข้อเสีย:
    ❌ เห็นใน Jaeger ช้าสุด 5 วิ (development ใจร้อนได้)
    ❌ ถ้า process ตายก่อน flush → spans สูญหาย

→ Development: SimpleSpanProcessor (เห็นทันที)
→ Production:  BatchSpanProcessor (ไม่ block, efficient)
```

---

## 9. ตัวอย่างข้อมูลจริงทุกขั้นตอน

### ขั้นที่ 1: SDK สร้าง Span ใน Memory

```go
// เมื่อเรียก tracer.Start(ctx, "Insert")
// SDK สร้าง recordingSpan struct (in-memory):

recordingSpan{
    // ─── Identity ───
    traceID:     [16]byte{0x4b, 0xf9, 0x2f, 0x35, 0x77, 0xb3, 0x4d, 0xa6,
                          0xa3, 0xce, 0x92, 0x9d, 0x0e, 0x0e, 0x47, 0x36},
    spanID:      [8]byte{0xbb, 0xbb, 0xbb, 0xbb, 0xbb, 0xbb, 0x00, 0x02},
    parentSpanID:[8]byte{0xaa, 0xaa, 0xaa, 0xaa, 0xaa, 0xaa, 0x00, 0x01},

    // ─── Info ───
    name:        "Insert",
    spanKind:    INTERNAL,
    startTime:   time.Time{2024-01-15 10:30:00.004},

    // ─── State (ยังเปิดอยู่) ───
    endTime:     time.Time{zero},   // ยังไม่ End
    status:      UNSET,

    // ─── Data ───
    attributes:  []KeyValue{},      // ยังว่าง
    events:      []Event{},         // ยังว่าง
    links:       []Link{},          // ยังว่าง
}
```

### ขั้นที่ 2: span.End() ถูกเรียก

```go
// SDK finalize span:

readOnlySpan{
    traceID:     "4bf92f3577b34da6a3ce929d0e0e4736",
    spanID:      "bbbbbbbbbbbb0002",
    parentSpanID:"aaaaaaaaaaaa0001",
    name:        "Insert",
    spanKind:    INTERNAL,
    startTime:   2024-01-15T10:30:00.004Z,
    endTime:     2024-01-15T10:30:00.074Z,   // ← set แล้ว
    duration:    70ms,                         // ← คำนวณแล้ว
    attributes: [
        {key: "company.id",   value: "cmp_abc123"},
        {key: "db.system",    value: "postgresql"},
        {key: "db.operation", value: "INSERT"},
    ],
    status: {code: OK},
    resource: {
        "service.name":    "sellsuki-central-control-backend-development",
        "service.version": "v0.0.0",
        "environment":     "development",
    },
}
// → ส่งเข้า BatchSpanProcessor.exportQueue channel
```

### ขั้นที่ 3: Serialize เป็น OTLP Protobuf

```
BatchSpanProcessor flush → Exporter serialize:

[OTLP ExportTraceServiceRequest]
  resource_spans:
    - resource:
        attributes:
          - key: "service.name"
            value: {string_value: "sellsuki-central-control-backend-development"}
          - key: "service.version"
            value: {string_value: "v0.0.0"}

      scope_spans:
        - scope:
            name: "company-repository"
          spans:
            - trace_id:        [4b f9 2f 35 77 b3 4d a6 a3 ce 92 9d 0e 0e 47 36]
              span_id:         [bb bb bb bb bb bb 00 02]
              parent_span_id:  [aa aa aa aa aa aa 00 01]
              name:            "Insert"
              kind:            SPAN_KIND_INTERNAL
              start_time_unix_nano: 1705312200004000000
              end_time_unix_nano:   1705312200074000000
              attributes:
                - key: "company.id"    value: "cmp_abc123"
                - key: "db.system"     value: "postgresql"
                - key: "db.operation"  value: "INSERT"
              status: {code: STATUS_CODE_OK}

Binary size: ~300 bytes per span
ส่งผ่าน gRPC ไปที่ localhost:4317
```

### ขั้นที่ 4: OTEL Collector รับและ Forward

```
Collector รับ OTLP batch:
  otlp receiver → pipeline
  → filter processor: ผ่าน (ไม่ใช่ /health)
  → batch processor: รวมกับ spans อื่น
  → otlp/jaeger exporter: ส่ง OTLP ไป Jaeger port 4317
  → logging exporter: print log

Log ที่เห็น:
  2024-01-15T10:30:05.100Z INFO Traces Exporter
    {
      "otelcol.exporter.otlp": {
        "spans": 2,
        "failed": 0
      }
    }
```

### ขั้นที่ 5: Jaeger เก็บและ Index

```
Jaeger รับ spans:
  → parse OTLP
  → เก็บลง storage (in-memory หรือ Cassandra/Elasticsearch)
  → index ด้วย traceId, serviceName, operationName, startTime, tags

In-Memory Storage (local development):
  map[TraceID][]Span{
    "4bf92f35...": [
      {spanId: "bb0002", parentId: "aa0001", name: "Insert", dur: 70ms},
      {spanId: "aa0001", parentId: "",       name: "HTTP POST /v1/companies", dur: 100ms},
    ]
  }
```

### ขั้นที่ 6: Jaeger UI Query และ Assembly

```
User ค้นหา → Jaeger UI → HTTP GET /api/traces?service=...

Jaeger backend:
  1. ค้นหา traceIds ที่ match service + time range
  2. ดึง spans ทั้งหมดของแต่ละ traceId
  3. sort spans by startTime
  4. สร้าง tree structure จาก parentId
  5. คำนวณ timeline, critical path
  6. ส่ง JSON กลับไป UI

JSON Response:
{
  "data": [{
    "traceID": "4bf92f3577b34da6a3ce929d0e0e4736",
    "spans": [
      {
        "traceID": "4bf92f35...",
        "spanID":  "aaaaaaaaaaaa0001",
        "operationName": "HTTP POST /v1/companies",
        "duration": 100000,  // microseconds
        "tags": [
          {"key": "http.method",      "value": "POST"},
          {"key": "http.status_code", "value": 201}
        ]
      },
      {
        "traceID": "4bf92f35...",
        "spanID":  "bbbbbbbbbbbb0002",
        "parentSpanID": "aaaaaaaaaaaa0001",
        "operationName": "Insert",
        "duration": 70000,
        "tags": [
          {"key": "company.id",   "value": "cmp_abc123"},
          {"key": "db.operation", "value": "INSERT"}
        ]
      }
    ],
    "processes": {
      "p1": {"serviceName": "sellsuki-central-control-backend-development"}
    }
  }]
}
```

---

## Quick Summary Cards

### TextMapCarrier vs TextMapPropagator

```
┌─────────────────────────────────────────────────────────────┐
│  TextMapCarrier         │  TextMapPropagator                │
│  ─────────────────────  │  ─────────────────────────────── │
│  คืออะไร: "กล่อง"       │  คืออะไร: "ตรรกะอ่าน/เขียน"    │
│  รู้อะไร: แค่ Get/Set   │  รู้อะไร: format traceparent    │
│  ตัวอย่าง:              │  ตัวอย่าง:                      │
│    HTTP Header          │    TraceContext propagator       │
│    Kafka Headers        │    Baggage propagator           │
│    Map[string]string    │    B3 propagator                │
│  ใช้ตอน:               │  ใช้ตอน:                        │
│    เป็น argument        │    ใช้ carrier เป็น tool        │
│    ให้ Propagator       │    inject/extract context       │
└─────────────────────────────────────────────────────────────┘
     ใช้ร่วมกันเสมอ: Propagator(ctx, Carrier)
```

### Span Lifecycle

```
tracer.Start()  → in-flight span (ใน memory เท่านั้น)
span.End()      → finalize → ส่ง BatchSpanProcessor
5 วิ ผ่านไป   → flush batch → OTEL Collector → Jaeger
```

### Jaeger Assembly

```
Service A, B, C ส่ง spans แยกกัน คนละเวลา
Jaeger รับมาเก็บไว้ก่อน
ตอน query ค่อย group ด้วย traceId
→ ไม่มีการรอให้ครบ, ไม่มี coordination ระหว่าง services
```
