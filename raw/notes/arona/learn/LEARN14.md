# 💻 เรียนรู้ Arona (ตอนที่ 14): การเชื่อมต่อกับ Frontend (API Consumption & Streaming)

ในบทสุดท้ายนี้ เราจะมาดูวิธีนำ Arona ไปใช้งานในหน้าเว็บจริงๆ ครับ เพื่อให้ User สามารถแชทได้เหมือน ChatGPT

---

## 1. การเรียกใช้ API (`POST /ai/ask`)
Request ที่ Frontend ต้องส่งมามีหน้าตาแบบนี้:

```json
{
  "message": "ระเบียบการลาป่วยเป็นอย่างไร?",
  "history": [
    { "role": "user", "content": "สวัสดี" },
    { "role": "assistant", "content": "สวัสดีค่ะ มีอะไรให้ช่วยไหมคะ?" }
  ],
  "think": false
}
```

---

## 2. การจัดการ Streaming (สำคัญมาก!)
Arona ใช้การตอบกลับแบบ **Streaming** (พ่นคำตอบออกมาทีละคำ) เพื่อให้ User รู้สึกว่าระบบตอบไว:
*   Frontend ต้องรองรับการอ่านข้อมูลแบบ `ReadableStream`
*   ใน JavaScript คุณสามารถใช้ `fetch` และอ่าน `response.body` ได้เลย

---

## 3. ส่วนประกอบของ Response
นอกจากข้อความ (Text) แล้ว Arona ยังส่งข้อมูลอื่นๆ กลับมาด้วย:
*   **References:** รายชื่อเอกสารที่ AI ใช้ตอบคำถาม (ควรเอาไปโชว์ใน UI เป็นแหล่งอ้างอิง)
*   **Duration:** เวลาที่ใช้ประมวลผลทั้งหมด

---

## 4. ตัวอย่างโค้ดฝั่ง Frontend (React/Next.js)
คุณสามารถใช้ไลบรารีอย่าง `ai/react` (จาก Vercel AI SDK) เพื่อให้เขียนโค้ดได้ง่ายขึ้น:

```javascript
import { useChat } from 'ai/react';

const ChatComponent = () => {
  const { messages, input, handleInputChange, handleSubmit } = useChat({
    api: '/api/ask', // ชี้ไปที่เซิร์ฟเวอร์ Arona
  });

  return (
    <div>
      {messages.map(m => (
        <div key={m.id}>{m.role}: {m.content}</div>
      ))}
      <form onSubmit={handleSubmit}>
        <input value={input} onChange={handleInputChange} />
        <button type="submit">ถาม Arona</button>
      </form>
    </div>
  );
};
```

---

## 5. การตั้งค่าความปลอดภัย (CORS)
ใน `src/server.ts` มีการตั้งค่า **CORS** ไว้เพื่อให้หน้าเว็บจาก Domain อื่นสามารถเรียกใช้ได้ อย่าลืมเพิ่ม Domain ของคุณเข้าไปในรายการอนุญาต:

```typescript
.use(cors({
    origin: ['http://localhost:5173', 'https://your-domain.com']
}))
```

---

## 🎉 บทสรุปของหลักสูตร Arona
ยินดีด้วยครับ! ตอนนี้คุณคือผู้เชี่ยวชาญ Arona ตัวจริงแล้ว คุณรู้วิธี:
1.  **Ingest** ข้อมูลให้สะอาดและประหยัด
2.  **Search** ข้อมูลให้แม่นยำด้วย Hybrid Search
3.  **Chat** ด้วย AI ที่ฉลาดและมีเหตุผล (Tools & Prompt)
4.  **Secure** ระบบด้วยระบบสิทธิ์และ Rate Limit
5.  **Scale** และวัดผลเพื่อให้ระบบเก่งขึ้นเรื่อยๆ

**"ขอให้สนุกกับการสร้างสรรค์สิ่งใหม่ๆ ด้วย Arona นะครับ!"**

---

**กลับไปหน้าหลัก [LEARN.MD](./LEARN.MD)**
