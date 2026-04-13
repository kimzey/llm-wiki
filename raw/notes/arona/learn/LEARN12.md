# 🛠️ เรียนรู้ Arona (ตอนที่ 12): การสร้างเครื่องมือใหม่ให้ AI (Custom Tools Development)

ถ้า Prompt คือ "ความคิด" Tools ก็คือ **"มือ"** ของ AI ครับ ในหน้านี้เราจะมาดูวิธีสร้างเครื่องมือใหม่ๆ ให้ Arona สามารถทำอย่างอื่นได้นอกจากค้นหาเอกสาร

---

## 1. โครงสร้างของ Tool
ใน `src/modules/ai/libs/tool.ts` เราใช้ฟังก์ชัน `tool()` เพื่อประกาศเครื่องมือ ซึ่งมี 3 ส่วนประกอบหลัก:
1.  **`description`**: อธิบายให้ AI ฟังว่าเครื่องมือนี้เอาไว้ทำอะไร (สำคัญมาก! AI จะใช้สิ่งนี้ตัดสินใจว่าจะเรียกใช้เครื่องมือนี้เมื่อไหร่)
2.  **`inputSchema`**: กำหนดหน้าตาข้อมูลที่ AI ต้องส่งมาให้ (ใช้ Zod)
3.  **`execute`**: โค้ดที่รันจริงๆ (ส่วนนี้แหละที่เราเขียน Logic อะไรก็ได้)

---

## 2. ตัวอย่างการสร้าง Tool ใหม่ (เช่น เช็คยอดเงินกองทุน)
สมมติคุณอยากให้ Arona สามารถเช็คยอดเงินกองทุนสำรองเลี้ยงชีพของพนักงานได้:

```typescript
export const checkFundBalanceTool = tool({
    description: "Check the current balance of the user's provident fund.",
    inputSchema: z.object({
        employee_id: z.string()
    }),
    async execute({ employee_id }) {
        // ในความเป็นจริง เราอาจจะไปเรียก API ภายนอก หรือดึงจาก DB
        const balance = await db.getBalance(employee_id);
        return { balance, currency: 'THB' };
    }
})
```

---

## 3. การนำไปใช้งาน (Registering the Tool)
หลังจากสร้างเสร็จแล้ว คุณต้องเอาไปใส่ในรายการ Tools ใน `src/modules/ai/service.ts`:

```typescript
// ในฟังก์ชัน ask()
const fundTool = checkFundBalanceTool;

return streamText({
    model,
    tools: {
        search: searchTool,
        readDocument: readDocTool,
        checkFund: fundTool // เพิ่มเข้าไปที่นี่!
    },
    // ...
})
```

---

## 4. เคล็ดลับในการสร้าง Tool ที่ดี
1.  **ชื่อ Tool ชัดเจน:** ใช้ชื่อที่สื่อความหมาย เช่น `getEmployeeStatus` ดีกว่า `toolA`
2.  **Description ละเอียด:** บอก AI ว่าควรเรียกใช้เครื่องมือนี้ตอนไหน เช่น *"Call this when the user asks about their financial status"*
3.  **Handle Errors:** ใน `execute` ควรมีการดักจับ Error และคืนค่าที่ AI เข้าใจได้ (เช่น `null` หรือข้อความ Error)

---

**สรุป:** การสร้าง Tool คือการขยายความสามารถให้ AI ของคุณทำงานได้มากกว่าแค่การแชท (เช่น เชื่อมต่อกับ DB อื่น, เรียก API ภายนอก, หรือแม้แต่สั่งรันสคริปต์)

**ไปต่อกันที่ [LEARN13.MD](./LEARN13.MD) เพื่อดูวิธีวัดผลว่าระบบที่เราทำมัน "แม่น" จริงไหม!**
