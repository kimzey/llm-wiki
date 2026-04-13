# LangFlow Custom Component — อธิบายทุกอย่างแบบละเอียด

---

## 1. นี่คืออะไร? LangChain หรือ LangFlow?

**ตอบ: ทั้งคู่ครับ**

| Layer | คืออะไร |
|---|---|
| **LangFlow** | GUI Builder — ลาก/วาง Block เชื่อมกัน (เหมือน n8n) |
| **LangChain** | Python Library — logic จริงๆ อยู่ที่นี่ |

LangFlow ใช้ LangChain อยู่ข้างใน  
`ChatOpenAI`, `RedisCache` ที่ import มา **คือ LangChain objects ทั้งนั้น**

---

## 2. `self.model_name`, `self.api_key` มาจากไหน?

### คำตอบ: มาจาก `inputs = [...]` ผ่าน Magic ของ `Component` base class

```python
# เมื่อคุณประกาศ Input แบบนี้:
inputs = [
    SecretStrInput(name="api_key", ...),   # ← name="api_key"
    DropdownInput(name="model_name", ...),  # ← name="model_name"
    StrInput(name="redis_url", ...),
    IntInput(name="cache_ttl", ...),
    IntInput(name="max_tokens", ...),
]
```

LangFlow's `Component` base class จะทำ **auto-binding** ให้อัตโนมัติ:

```
name="api_key"      →  self.api_key
name="model_name"   →  self.model_name
name="redis_url"    →  self.redis_url
name="cache_ttl"    →  self.cache_ttl
name="max_tokens"   →  self.max_tokens
```

**กฎ: ค่าของ `name=` จะกลายเป็น attribute ของ `self` เสมอ**

---

## 3. Flow ชีวิตของ Component

```
User กรอก UI
      ↓
LangFlow เก็บค่าใน Input objects
      ↓
Component.__init__() bind ค่าเข้า self.xxx
      ↓
LangFlow เรียก build_model()
      ↓
คุณใช้ self.api_key, self.model_name ฯลฯ ได้เลย
      ↓
return ChatOpenAI(...)  ← ส่งออกไปที่ Output port
```

---

## 4. แต่ละ Input Type คืออะไร?

| Class | ชนิดข้อมูล | พิเศษ |
|---|---|---|
| `StrInput` | `str` | ธรรมดา |
| `SecretStrInput` | `str` | ซ่อนใน UI (password field) |
| `IntInput` | `int` | ตัวเลขจำนวนเต็ม |
| `DropdownInput` | `str` | Dropdown ให้เลือก |
| `BoolInput` | `bool` | True/False |
| `FloatInput` | `float` | ทศนิยม |
| `MessageTextInput` | `str` | รับข้อความจาก Chat |

---

## 5. Output คืออะไร?

```python
outputs = [
    Output(
        display_name="Language Model",  # ชื่อที่แสดงใน UI
        name="model",                   # ชื่อ port
        method="build_model",           # ← เรียก method นี้เมื่อ run
    ),
]
```

เมื่อ LangFlow รัน → มันจะเรียก `self.build_model()` → เอาผลลัพธ์ส่งออก port "model"

---

## 6. ส่วน Redis Cache ทำงานยังไง?

```python
def build_model(self) -> ChatOpenAI:
    # Step 1: เชื่อม Redis
    r = redis.from_url(self.redis_url)

    # Step 2: ผูก cache เข้ากับ LangChain แบบ global
    langchain.llm_cache = RedisCache(
        redis_=r,
        ttl=self.cache_ttl if self.cache_ttl > 0 else None,
    )
    # ตอนนี้ทุก LLM call ใน process นี้จะใช้ Redis cache

    # Step 3: สร้าง ChatOpenAI ที่ชี้ไป OpenRouter
    return ChatOpenAI(
        model=self.model_name,
        openai_api_key=self.api_key,
        openai_api_base="https://openrouter.ai/api/v1",  # ← ใช้ OpenRouter แทน OpenAI
        max_tokens=self.max_tokens,
        cache=True,  # ← บอกให้ใช้ cache
    )
```

**Cache Flow:**
```
Agent ถามคำถาม
      ↓
LangChain เช็ค Redis ก่อน
      ↓ (cache hit)          ↓ (cache miss)
return คำตอบเลย         ส่งไป OpenRouter API
                               ↓
                         เก็บผลลัพธ์ใน Redis
                               ↓
                         return คำตอบ
```

---

## 7. Template สำหรับเขียน Component ใหม่

```python
from lfx.custom.custom_component.component import Component
from lfx.io import Output, StrInput, IntInput, BoolInput, SecretStrInput
from lfx.inputs.inputs import DropdownInput

class MyComponent(Component):
    display_name = "ชื่อที่แสดงใน UI"
    description = "อธิบายสั้นๆ"
    icon = "ชื่อ icon (lucide)"

    inputs = [
        # แต่ละ input → กลายเป็น self.<name> อัตโนมัติ
        SecretStrInput(
            name="my_secret",       # → self.my_secret
            display_name="API Key",
            required=True,
        ),
        StrInput(
            name="my_text",         # → self.my_text
            display_name="Text",
            value="default value",
        ),
        IntInput(
            name="my_number",       # → self.my_number
            display_name="Number",
            value=100,
            advanced=True,          # ซ่อนใน Advanced section
        ),
        DropdownInput(
            name="my_choice",       # → self.my_choice
            display_name="เลือก",
            options=["a", "b", "c"],
            value="a",
        ),
    ]

    outputs = [
        Output(
            display_name="ผลลัพธ์",
            name="result",
            method="build_result",  # ← ชื่อ method ที่จะเรียก
        ),
    ]

    def build_result(self):
        # ใช้ self.my_secret, self.my_text ฯลฯ ได้เลย
        self.log(f"Running with: {self.my_text}")
        
        result = do_something(
            key=self.my_secret,
            text=self.my_text,
            n=self.my_number,
        )
        return result
```

---

## 8. สรุปกฎที่ต้องจำ

1. **`name=` ใน inputs → `self.<name>` ใน method** — เสมอ ไม่มีข้อยกเว้น
2. **`method=` ใน outputs** — ชี้ไปหา method ที่จะ return ค่าออก
3. **return type** — ควรระบุให้ตรงกับที่ port คาดหวัง (เช่น `ChatOpenAI`, `str`, `list`)
4. **`self.log()`** — ใช้ debug แทน `print()` ใน LangFlow
5. **`advanced=True`** — ซ่อน input ไว้ใน "Advanced" ไม่รก UI
