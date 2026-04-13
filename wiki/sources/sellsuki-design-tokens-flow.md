---
title: "Sellsuki Design Tokens — Flow การทำงาน"
type: source
source_file: raw/notes/central-component/design-tokens-flow.md
url: ""
published: 2026-03-01
tags: [design-tokens, css-variables, typescript, flow, sellsuki]
related: [wiki/sources/sellsuki-design-tokens-service.md, wiki/concepts/design-tokens.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/central-component/design-tokens-flow.md|Original file]]

## สรุป

อธิบาย flow การทำงานของ Design Tokens แบบละเอียด เน้น separation of concerns ระหว่าง CSS Variables (สำหรับ browser) กับ TypeScript Types (สำหรับ developer) พร้อม diagram, checklist, และ FAQ

## ประเด็นสำคัญ

- **Flow หลักสั้นๆ**: `design-tokens.css → global.css → main.ts → Browser`
- **Separation of Concerns**: CSS Variables สำหรับ runtime, TypeScript Types สำหรับ devtime — คนละ layer คนละหน้าที่
- **`theme.ts` ไม่มี CSS Variables** เพราะ CSS Variables ต้องอยู่ใน `.css` ไม่ใช่ JavaScript — ถ้าอยู่ใน JS ต้อง inject เข้า DOM เอง (ซับซ้อนกว่า)
- **Two systems must match**: ถ้าเพิ่ม token ใน CSS ต้องเพิ่ม type ใน `theme.ts` ด้วย — แต่ไม่มี automatic sync
- **cssVar() helper**: generate ชื่อ CSS variable — `cssVar('spacing', 'md')` → `"--ssk-spacing-md"`

## ข้อมูล / หลักฐาน ที่น่าสนใจ

**Checklist เมื่อเพิ่ม Token ใหม่:**
1. เพิ่มใน `design-tokens.css` → `--ssk-spacing-huge: var(--ssk-space-96);`
2. เพิ่มใน `theme.ts` → `export type SpacingSize = ... | "huge";`
3. Rebuild → `npm run build`

**ทดสอบ CSS variables ใน Browser DevTools:**
```javascript
getComputedStyle(document.documentElement)
  .getPropertyValue('--ssk-spacing-md')
// ผลลัพธ์: "var(--ssk-space-6)" หรือ "0.375rem"
```

**ทำไมไม่รวมทุกอย่างใน `theme.ts`?**
> CSS Variables ต้องอยู่ใน CSS ไม่ใช่ JavaScript — ถ้าอยู่ใน JavaScript ต้อง inject เข้า DOM (ซับซ้อนกว่า)

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่มีข้อขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/design-tokens.md|Design Tokens]]
