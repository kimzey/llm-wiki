# Distributed Tracing & Context Propagation

> เนื้อหาครอบคลุมตั้งแต่พื้นฐานจนถึงรายละเอียดของ
> `otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(...))`

---

## 1. ภาพรวม — ทำไมต้องมีเรื่องนี้?

### ระบบเดิม (Monolith)

```
User → [ Application เดียว ] → Database
```

ถ้า error เกิด → ดู log ใน process เดียวได้เลย

### ระบบปัจจุบัน (Distributed System / Microservices)

```
User Request
    │
    ▼
Central Control Backend  ──gRPC──►  Role And Permission Service
    │                                         │
    ▼                                         ▼
Order Service  ──HTTP──►  Payment Service    Keto (AuthZ)
```

ถ้า error เกิด หรือ response ช้า → ไม่รู้ว่า **request เดิมใบนั้น** เดินทางผ่านไหนบ้าง
เพราะแต่ละ service อยู่คนละ process คนละ machine คนละ log file

**ปัญหา:** request 1 ใบ ทำให้เกิด log กระจายอยู่ใน 4-5 service
**คำถาม:** log พวกนั้นเป็นของ request เดียวกันมั้ย?

---

## 2. Distributed Tracing คืออะไร?

**Distributed Tracing = ระบบติดตาม request เดียวข้ามหลาย service**

เหมือนการติด **GPS tracker** ไว้กับ request ตั้งแต่ต้นจนจบ
ทุก service ที่ request ผ่าน จะบันทึกว่า "ฉันรับ request นี้ไว้ ใช้เวลา X ms"

### ศัพท์สำคัญ

| คำ | ความหมาย | ตัวอย่าง |
|---|---|---|
| **Trace** | การเดินทางทั้งหมดของ 1 request | TraceID: `b71df886bfbb3cca` |
| **Span** | งานชิ้นเดียวใน 1 service | SpanID: `9c4a6dbf93d0594d` |
| **Parent Span** | span ที่เรียก span อื่น | Central Control เรียก Role Permission |
| **Child Span** | span ที่ถูกเรียก | Role Permission ถูกเรียกจาก Central Control |
| **TraceID** | เลข ID ของ trace ทั้งหมด | เหมือนเลขพัสดุ |
| **SpanID** | เลข ID ของ span นั้นๆ | เหมือนเลข tracking ของแต่ละด่าน |

### ผลลัพธ์ที่เห็นใน Jaeger / Grafana Tempo

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

---

## 3. Context Propagation คืออะไร?

**Context Propagation = กลไกส่ง "ตัวตน" ของ request ข้าม service**

แต่ละ service อยู่คนละ process → memory ไม่แชร์กัน
ต้องส่ง TraceID/SpanID ผ่าน **network header** เท่านั้น

### Analogy: เลขพัสดุ

```
ไปรษณีย์ A (Central Control)
────────────────────────────────
มีพัสดุ เลขที่: TH1234567890
อยากให้ทุกด่านรู้ว่านี่คือพัสดุใบเดียวกัน

ขั้นตอน:
1. เขียนเลขพัสดุลงบน "ฉลาก" กล่อง     ← Inject
2. ส่งกล่องไปยังด่านถัดไป
3. ด่านถัดไปอ่านฉลาก                   ← Extract
4. รู้ว่าเลขพัสดุคืออะไร → ต่อ trace ได้
```

**ฉลาก = network header**
**เลขพัสดุ = TraceID**

---

## 4. Propagator คืออะไร?

**Propagator = "ล่าม" ที่แปลงระหว่าง context ใน memory ↔ text header บน network**

คำว่า propagate = แพร่กระจาย / ส่งต่อ

Propagator ทำ **2 หน้าที่เท่านั้น**:

```
Inject  → อ่าน context จาก memory → เขียนลง header
Extract → อ่าน header → สร้าง context ขึ้นใน memory
```

### Propagator ไม่ใช่ "ชนิดการส่งข้อมูล"

มันไม่เกี่ยวกับ gRPC vs HTTP vs Kafka — นั่นคือ transport layer คนละเรื่องกัน

```
┌──────────────────────────────────────────────┐
│  Application Layer                           │
│  TraceID, SpanID, Baggage อยู่ใน memory      │
├──────────────────────────────────────────────┤
│  Propagator Layer          ← ตรงนี้         │
│  แปลง context ↔ text header                 │
│  ไม่สนว่าจะส่งผ่าน protocol อะไร            │
├──────────────────────────────────────────────┤
│  Transport Layer                             │
│  gRPC metadata / HTTP header / Kafka header  │
└──────────────────────────────────────────────┘
```

Propagator แค่บอกว่า **"เขียนข้อมูลลง field อะไร ในรูปแบบไหน"**

---

## 5. TextMapPropagator คืออะไร?

**TextMapPropagator = propagator ที่ทำงานกับ text key-value pairs**

"TextMap" = map ของ string key → string value
ซึ่งตรงกับ HTTP headers และ gRPC metadata พอดี

```go
// interface ของ TextMapPropagator
type TextMapPropagator interface {
    Inject(ctx context.Context, carrier TextMapCarrier)
    Extract(ctx context.Context, carrier TextMapCarrier) context.Context
    Fields() []string // บอกว่าใช้ header อะไรบ้าง
}
```

**TextMapCarrier** = ตัวกลางที่ hold headers (gRPC metadata, HTTP header, etc.)

---

## 6. ทำความเข้าใจ Code ทีละบรรทัด

```go
otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
    propagation.TraceContext{},
    propagation.Baggage{},
))
```

---

### 6.1 `otel.SetTextMapPropagator(...)`

ตั้งค่า **global propagator** ให้ทั้ง process

```go
// ก่อนเรียก → propagator เป็น noop (ไม่ทำอะไร)
// otelgrpc inject/extract → ไม่มี header → trace ขาด

otel.SetTextMapPropagator(myPropagator)

// หลังเรียก → otelgrpc ใช้ myPropagator inject/extract อัตโนมัติ
```

ทำไมต้องเป็น global? เพราะ `otelgrpc`, `otelhttp` และ lib อื่นๆ ทุกตัว
เรียก `otel.GetTextMapPropagator()` เพื่อดึง propagator ไปใช้
→ ถ้าไม่ set → ได้ noop กลับมา → ไม่มีอะไรเกิดขึ้น

---

### 6.2 `propagation.NewCompositeTextMapPropagator(...)`

รวมหลาย propagator ให้ทำงานเป็นตัวเดียว

```go
// การทำงานตอน Inject
composite.Inject(ctx, carrier):
    propagator1.Inject(ctx, carrier)  // เขียน traceparent header
    propagator2.Inject(ctx, carrier)  // เขียน baggage header
    // ...ทำทีละตัวตามลำดับ

// การทำงานตอน Extract
composite.Extract(ctx, carrier):
    ctx = propagator1.Extract(ctx, carrier)  // อ่าน traceparent
    ctx = propagator2.Extract(ctx, carrier)  // อ่าน baggage
    // ...ทำทีละตัว context ถูก enrich ทีละชั้น
```

---

### 6.3 `propagation.TraceContext{}`

ใช้ **W3C Trace Context standard** (RFC W3C 2020)

**หน้าที่:** inject/extract TraceID และ SpanID ผ่าน header `traceparent`

#### รูปแบบ header traceparent

```
traceparent: 00-b71df886bfbb3cca29211a10b74c6212-9c4a6dbf93d0594d-01
             │  │                                │                │
             │  │                                │                └── flags (01 = sampled)
             │  │                                └── Parent SpanID (64-bit hex = 16 chars)
             │  └── TraceID (128-bit hex = 32 chars)
             └── version (00 = current version)
```

#### ตัวอย่างจากโปรเจกต์นี้จริงๆ

```
// Central Control ส่งมา:
traceparent: 00-b71df886bfbb3cca29211a10b74c6212-9c4a6dbf93d0594d-01

// Role Permission อ่านแล้วสร้าง context:
TraceID:      b71df886bfbb3cca29211a10b74c6212  ← เอาจาก header
ParentSpanID: 9c4a6dbf93d0594d                  ← SpanID ของ Central Control
SpanID:       fa3c19e820b74a12                  ← สร้างใหม่ (child span)
```

#### flags byte หมายความว่าอะไร?

```
00 = ไม่ sample (ไม่ส่ง trace ไป backend)
01 = sampled   (ส่ง trace ไป Jaeger/Tempo)
```

#### ทำไมใช้ W3C?

เพราะเป็น standard สากลที่ทุก vendor รองรับ:
- Jaeger ✅
- Grafana Tempo ✅
- Datadog ✅
- Zipkin ✅
- AWS X-Ray ✅ (รองรับควบคู่กับ format ของตัวเอง)

---

### 6.4 `propagation.Baggage{}`

ใช้ **W3C Baggage standard**

**หน้าที่:** inject/extract key-value metadata ผ่าน header `baggage`

#### รูปแบบ header baggage

```
baggage: userId=user-123,tenantId=acme-corp,featureFlag=new-ui
         │               │                  │
         key=value        key=value          key=value (คั่นด้วย comma)
```

#### ต่างจาก TraceContext ยังไง?

```
TraceContext → ส่ง "ตัวตน" ของ trace
               ใช้เพื่อ connect spans ให้เป็น tree
               TraceID, SpanID, flags

Baggage      → ส่ง "ข้อมูล" เพิ่มเติม
               ใช้เพื่อส่ง context ข้อมูลข้าม service
               key-value อะไรก็ได้ที่ต้องการ
```

#### ตัวอย่างการใช้ Baggage

```go
// Service A: ใส่ข้อมูลลง baggage
b, _ := baggage.Parse("userId=user-123,tenantId=acme-corp")
ctx = baggage.ContextWithBaggage(ctx, b)
// → otelgrpc inject header: baggage: userId=user-123,tenantId=acme-corp

// Service B, C, D รับมาอัตโนมัติผ่าน propagator
b := baggage.FromContext(ctx)
userId := b.Member("userId").Value()   // "user-123"
tenant := b.Member("tenantId").Value() // "acme-corp"
```

ใช้เมื่อ: อยากส่ง metadata ข้าม service โดยไม่ต้องแก้ function signature ทุกชั้น

#### service นี้ใช้ Baggage มั้ย?

ไม่ใช้ครับ — ค้นหา `baggage` ใน codebase ไม่เจอเลย
แต่ใส่ไว้ได้เพราะ `propagation.Baggage{}` เป็น struct เปล่า ขนาด 0 bytes
ตอน runtime ถ้าไม่มี baggage header → return ทันที ไม่กิน resource

---

## 7. Flow การทำงานจริง Step by Step

### ก่อนแก้ (ไม่มี propagator)

```
Central Control Backend
────────────────────────────────────────────────────────
1. สร้าง Span
   TraceID: b71df886bfbb3cca29211a10b74c6212
   SpanID:  9c4a6dbf93d0594d

2. otelgrpc พยายาม inject header
   → เรียก otel.GetTextMapPropagator() → ได้ noop กลับมา
   → noop.Inject() → ไม่เขียน header อะไรเลย

3. ส่ง gRPC request → ไม่มี traceparent header

Role And Permission Backend
────────────────────────────────────────────────────────
4. otelgrpc พยายาม extract header
   → เรียก otel.GetTextMapPropagator() → ได้ noop กลับมา
   → noop.Extract() → ไม่อ่าน header อะไรเลย

5. สร้าง Span ใหม่โดยไม่มี parent
   TraceID: e5f9ef4ec987fe81da2b46130829550a  ← คนละ trace!
   SpanID:  da2b46130829550a
```

**ผลลัพธ์:** Jaeger เห็น 2 trace แยกกัน ไม่รู้ว่า request เดียวกัน

---

### หลังแก้ (มี propagator)

```
Central Control Backend
────────────────────────────────────────────────────────
1. SetTextMapPropagator(TraceContext{}) ถูกเรียกตอน init

2. สร้าง Span
   TraceID: b71df886bfbb3cca29211a10b74c6212
   SpanID:  9c4a6dbf93d0594d

3. otelgrpc เรียก propagator.Inject()
   TraceContext.Inject() เขียน header:
   traceparent: 00-b71df886bfbb3cca29211a10b74c6212-9c4a6dbf93d0594d-01

4. ส่ง gRPC request พร้อม header

                    ↓↓↓ network ↓↓↓

Role And Permission Backend
────────────────────────────────────────────────────────
5. otelgrpc รับ request → เห็น traceparent header

6. เรียก propagator.Extract()
   TraceContext.Extract() อ่าน header:
   traceparent: 00-b71df886bfbb3cca29211a10b74c6212-9c4a6dbf93d0594d-01

7. สร้าง context ใหม่:
   TraceID:      b71df886bfbb3cca29211a10b74c6212  ← เอาจาก header (ไม่สร้างใหม่!)
   ParentSpanID: 9c4a6dbf93d0594d                  ← SpanID ของ Central Control
   SpanID:       fa3c19e820b74a12                  ← สร้างใหม่ (child span ของตัวเอง)
```

**ผลลัพธ์:** Jaeger เห็น 1 trace ที่มี span ต่อกันเป็น tree

---

## 8. ทำไม lib ไม่ set propagator ให้อัตโนมัติ?

### หลักการ: "No global side effects from import"

ถ้า otelgrpc set propagator ให้อัตโนมัติตอน import:

```go
// scenario ที่จะพัง
import _ "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
// → otelgrpc แอบ set propagator เป็น W3C

// แต่ developer ตั้งใจจะใช้ B3 (เพราะ infra เก่าใช้ Zipkin)
otel.SetTextMapPropagator(b3.New())
// → otelgrpc override ทับ! → ใช้ W3C แทน B3 → trace ขาด
```

### หลาย format อยู่ร่วมกันในโลกจริง

| Format | Header | ใช้ที่ไหน |
|---|---|---|
| W3C TraceContext | `traceparent` | standard ปัจจุบัน ทุก vendor |
| B3 Single | `b3` | Zipkin, Jaeger รุ่นเก่า |
| B3 Multi | `X-B3-TraceId` + `X-B3-SpanId` + ... | Zipkin legacy |
| Jaeger | `uber-trace-id` | Jaeger native format |
| AWS X-Ray | `X-Amzn-Trace-Id` | AWS services |

**ข้อมูลเดียวกัน แต่เขียนต่างกัน → ต้องตกลงกันทั้ง chain**

```
Central Control ──W3C──► Role Permission ──W3C──► Keto
     ✅ พูดภาษาเดียวกัน → trace ต่อกัน

Central Control ──B3───► Role Permission ──W3C──► Keto
     ❌ ต่างภาษา → extract ไม่ได้ → trace ขาด
```

---

## 9. ทำไมเลือกใส่หมดไม่ได้ / ควรทำมั้ย?

### ทำได้ แต่มีผลข้างเคียง

```go
otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
    propagation.TraceContext{}, // inject: traceparent
    propagation.Baggage{},      // inject: baggage
    b3.New(),                   // inject: b3 (X-B3-*)
    jaeger.Jaeger{},            // inject: uber-trace-id
))
```

**ผลที่เกิด:** ทุก request จะมี header เพิ่มขึ้น

```
// request ออกไปจะมี header ครบทุก format
traceparent:  00-b71df886...-9c4a6dbf...-01
baggage:      (ถ้ามี)
X-B3-TraceId: b71df886bfbb3cca29211a10b74c6212
X-B3-SpanId:  9c4a6dbf93d0594d
X-B3-Sampled: 1
uber-trace-id: b71df886bfbb3cca29211a10b74c6212:9c4a6dbf93d0594d:0:1
```

| ปัญหา | รายละเอียด |
|---|---|
| Header เยอะเกินจำเป็น | bandwidth เพิ่ม ทุก request |
| Dependency เพิ่ม | ต้อง import b3, jaeger lib เพิ่ม |
| ไม่มีประโยชน์ | ถ้าทั้ง chain ใช้ W3C อยู่แล้ว |

**แนะนำ:** เลือกแค่ format ที่ chain ทั้งหมดใช้จริง → สำหรับ project นี้คือ `TraceContext{}` + `Baggage{}` พอแล้ว

---

## 10. Component ทั้งหมดที่ต้องมีครบ

### 3 ส่วนที่ต้องทำงานร่วมกัน

```
┌─────────────────────────────────────────────────────────────────┐
│                         OTel Stack                              │
├─────────────────┬──────────────────────────┬────────────────────┤
│ TracerProvider  │   TextMapPropagator       │  otelgrpc Handler  │
│                 │                           │                    │
│ ส่ง trace data  │   แปลง context ↔ header   │  hook เข้า gRPC    │
│ ไปที่ backend   │   (inject / extract)      │  lifecycle         │
│ (Jaeger/Tempo)  │                           │                    │
└─────────────────┴──────────────────────────┴────────────────────┘
```

ขาดส่วนใดส่วนหนึ่ง → ระบบทำงานไม่ครบ

---

### ส่วนที่ 1: TracerProvider

บอกว่า trace data จะถูกส่งไปที่ไหน

```go
// helper.go
func initTracer(cfg config) {
    // สร้าง exporter → ส่งข้อมูลไป OTel Collector
    client := otlptracegrpc.NewClient(
        otlptracegrpc.WithInsecure(),
        otlptracegrpc.WithEndpoint(cfg.Services.OtelGrpcEndpoint),
    )
    exporter, _ := otlptrace.New(context.Background(), client)

    // สร้าง TracerProvider พร้อม metadata
    tp := trace.NewTracerProvider(
        trace.WithBatcher(exporter),
        trace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceName("role-permission-staging"),
            semconv.ServiceVersion("1.0.0"),
            attribute.String("environment", "staging"),
        )),
    )

    // ตั้งเป็น global
    otel.SetTracerProvider(tp)
}
```

---

### ส่วนที่ 2: TextMapPropagator (ตัวเอก)

บอกว่าจะ inject/extract header แบบไหน

```go
// helper.go (ต่อจาก SetTracerProvider)
otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
    propagation.TraceContext{}, // W3C standard → header "traceparent"
    propagation.Baggage{},      // metadata     → header "baggage"
))
```

**ต้องเรียกหลัง SetTracerProvider เสมอ** — เป็น global setting เช่นกัน

---

### ส่วนที่ 3: otelgrpc StatsHandler

hook เข้า gRPC lifecycle → เรียก propagator inject/extract อัตโนมัติ

```go
// grpc_server.go — ฝั่ง Server
grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()),
    // ...interceptors อื่นๆ
)

// helper.go — ฝั่ง Client (ตอน dial ไปหา service อื่น)
grpc.NewClient(
    address,
    grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

**ทำไมใช้ StatsHandler แทน Interceptor?**

```
Interceptor → จับได้แค่ unary/streaming call
              ไม่เห็น metadata, connection events

StatsHandler → จับได้ทุก event ใน gRPC lifecycle
               - Begin / End of RPC
               - InHeader / OutHeader (เห็น traceparent header!)
               - InPayload / OutPayload
               - ConnBegin / ConnEnd
```

StatsHandler เห็น header → extract traceparent → สร้าง child span ได้ถูกต้อง

---

## 11. ตัวอย่าง Request จริงในโปรเจกต์นี้

### สถานการณ์: Central Control เรียก ListRoles

```
Central Control Backend (มี otelgrpc.NewClientHandler())
──────────────────────────────────────────────────────────

กำลัง handle request ของ User อยู่ใน span:
  TraceID: b71df886bfbb3cca29211a10b74c6212
  SpanID:  9c4a6dbf93d0594d
  is_sampled: true ← log ที่เห็นบอกว่า is_sampled=true

ก่อน gRPC call → otelgrpc.NewClientHandler() inject:
  traceparent: 00-b71df886bfbb3cca29211a10b74c6212-9c4a6dbf93d0594d-01

                      ↓↓↓ gRPC call ↓↓↓

Role And Permission Service (มี otelgrpc.NewServerHandler())
──────────────────────────────────────────────────────────

รับ request → StatsHandler เห็น InHeader event
otelgrpc.NewServerHandler() extract:
  traceparent: 00-b71df886bfbb3cca29211a10b74c6212-9c4a6dbf93d0594d-01

สร้าง child span:
  TraceID:      b71df886bfbb3cca29211a10b74c6212  ← ตรงกัน!
  ParentSpanID: 9c4a6dbf93d0594d
  SpanID:       fa3c19e820b74a12                  ← ใหม่ (ของตัวเอง)

ListRoles handler ถูกเรียก → fmt.Printf:
  TraceID: b71df886bfbb3cca29211a10b74c6212       ← ตรงกับ Central Control!
```

---

## 12. Debug TraceID ใน Handler

```go
import "go.opentelemetry.io/otel/trace"

func (r RoleAndPermissionServiceServerImplementation) ListRoles(
    ctx context.Context,
    request *ListRolesRequest,
) (*ListRolesResponse, error) {

    // ดึง span จาก context
    span := trace.SpanFromContext(ctx)
    sc := span.SpanContext()

    fmt.Printf("TraceID: %s\n", sc.TraceID().String())
    fmt.Printf("SpanID: %s\n", sc.SpanID().String())
    fmt.Printf("IsValid: %v\n", sc.IsValid())   // false = ไม่มี trace
    fmt.Printf("IsSampled: %v\n", sc.IsSampled()) // false = ไม่ส่งไป backend

    // ...
}
```

---

## 13. Checklist สำหรับ Service ใหม่

- [ ] **TracerProvider** — ตั้งค่า exporter และ service metadata
- [ ] **TextMapPropagator** — set หลัง TracerProvider เสมอ
- [ ] **gRPC Server** — ใส่ `grpc.StatsHandler(otelgrpc.NewServerHandler())`
- [ ] **gRPC Client** — ใส่ `grpc.WithStatsHandler(otelgrpc.NewClientHandler())` ทุก connection
- [ ] **Format ตรงกัน** — ทั้ง chain ใช้ propagator format เดียวกัน
- [ ] **ทดสอบ** — `span.SpanContext().IsValid()` ต้องได้ `true`

---

## 14. สรุปภาพรวมทั้งหมด

```
otel.SetTextMapPropagator(
    propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},   propagation.Baggage{}
    )
)
       │
       │ ตั้ง global propagator
       ▼
┌──────────────────────────────────────────────────────┐
│  otelgrpc.NewClientHandler()  (ฝั่ง Client)          │
│                                                      │
│  ก่อนส่ง request → propagator.Inject()              │
│  เขียน: traceparent + baggage headers               │
└──────────────────────────┬───────────────────────────┘
                           │ gRPC call + headers
                           ▼
┌──────────────────────────────────────────────────────┐
│  otelgrpc.NewServerHandler()  (ฝั่ง Server)          │
│                                                      │
│  รับ request → propagator.Extract()                 │
│  อ่าน: traceparent → สร้าง child span               │
│  TraceID เดิม + SpanID ใหม่                         │
└──────────────────────────────────────────────────────┘
       │
       │ context พร้อมใช้งาน
       ▼
   Handler function (ctx มี TraceID ที่ต่อกับ caller)
```

| Component | หน้าที่ | ผลถ้าขาด |
|---|---|---|
| `SetTextMapPropagator` | global propagator | noop → ไม่มีการ inject/extract |
| `TraceContext{}` | inject/extract `traceparent` | trace ขาด → TraceID ไม่ตรงกัน |
| `Baggage{}` | inject/extract `baggage` | metadata ข้าม service ไม่ได้ |
| `NewServerHandler` | extract ฝั่ง server | server ไม่รู้จัก trace ของ caller |
| `NewClientHandler` | inject ฝั่ง client | ไม่ส่ง header → server ไม่มีอะไร extract |
