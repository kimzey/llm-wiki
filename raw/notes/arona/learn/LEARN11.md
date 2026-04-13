# ✍️ เรียนรู้ Arona (ตอนที่ 11): การปรับแต่งบุคลิกภาพและคำสั่ง AI (Prompt Engineering)

ในตอนนี้ เราจะมาดู "หัวใจ" ที่กำหนดพฤติกรรมของ Arona กันครับ ว่าทำไมมันถึงตอบแบบนั้น และเราจะเปลี่ยนมันได้อย่างไร

---

## 1. รู้จักกับ `instruction`
ในไฟล์ `src/libs/ai.ts` มีตัวแปรที่ชื่อว่า `instruction` ซึ่งเปรียบเสมือน **"กฎเหล็ก"** ที่ AI ต้องปฏิบัติตาม:
*   **Role:** บอกว่า AI คือใคร (Assistant ของ Sellsuki)
*   **Strict Rules:** กฎข้อห้ามต่างๆ (เช่น ห้ามมโนข้อมูลเอง, ต้องบอกที่มาเสมอ)
*   **Handle No Info:** กำหนดประโยคมาตรฐานเมื่อหาคำตอบไม่เจอ

---

## 2. เทคนิคการปรับแต่ง (How to customize)
หากคุณต้องการเปลี่ยน Arona ให้เป็นแบบที่คุณต้องการ:

### ก. เปลี่ยนบุคลิกภาพ (Personality)
ลองเปลี่ยนประโยคแรกจาก "You are a professional assistant..." เป็น:
*   *"You are a friendly and energetic office buddy..."* (เพื่อให้ AI ตอบแบบเป็นกันเองขึ้น)
*   *"You are a strict legal advisor..."* (เพื่อให้ AI ตอบแบบทางการและรัดกุมมาก)

### ข. เพิ่มกฎเฉพาะทาง (Specific Rules)
คุณสามารถเพิ่มกฎใหม่เข้าไปใน `STRICT RULES` ได้ เช่น:
*   *"Always provide answers in a bullet-point format."* (เพื่อให้ AI ตอบเป็นข้อๆ เสมอ)
*   *"If the user asks about money, remind them to consult with Finance department."*

---

## 3. ความสำคัญของ "No Hallucination"
กฎที่สำคัญที่สุดใน Arona คือ **"ห้ามมโน"**
*   เราสั่งให้ AI ตอบเฉพาะจากเอกสารที่หาเจอเท่านั้น (`Source-Only Answer`)
*   นี่คือหัวใจของ RAG เพราะเราต้องการความถูกต้อง 100% ไม่ใช่แค่คำตอบที่ฟังดูดีแต่ผิด

---

## 4. การจัดการภาษา (Bilingual)
Arona ถูกตั้งค่าให้ตอบตามภาษาที่ User ถามมา (`Answer in the same language as the user's question`) 
*   ถ้าคุณอยากให้ AI ตอบเป็นภาษาไทยเท่านั้น คุณสามารถแก้กฎข้อ 6 ให้เป็น *"Always answer in Thai language only"* ได้

---

**สรุป:** การแก้ `instruction` คือวิธีที่เร็วที่สุดในการเปลี่ยนพฤติกรรมของ AI โดยไม่ต้องแก้โค้ดส่วนอื่นเลย

**ไปต่อกันที่ [LEARN12.MD](./LEARN12.MD) เพื่อดูวิธีสร้าง "มือ" (Tools) ใหม่ๆ ให้ AI!**
