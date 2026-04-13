# Design Tokens - Flow การทำงานอธิบายแบบละเอียด

## 🔍 สรุปสั้นๆ ก่อน

```
design-tokens.css  →  global.css  →  main.ts  →  Browser
(ประกาศตัวแปร)      (รวม CSS)        (Bundle)        (ใช้งาน)
```

---

## 📂 โครงสร้างไฟล์และหน้าที่

### 1️⃣ `design-tokens.css` - ประกาศ CSS Variables

**ตำแหน่ง:** `src/assets/design-tokens.css`

**หน้าที่:** ประกาศ CSS Custom Properties (`--ssk-*`) ทั้งหมด

```css
/* src/assets/design-tokens.css */
:root {
  /* ประกาศตัวแปรไว้ที่นี่ */
  --ssk-space-8: 0.5rem;
  --ssk-spacing-md: var(--ssk-space-6);
  --ssk-radius-lg: var(--ssk-space-12);
}
```

> **สำคัญ:** ไฟล์นี้มีแค่ CSS ไม่มี JavaScript/TypeScript

---

### 2️⃣ `global.css` - รวม CSS ทั้งหมด

**ตำแหน่ง:** `src/assets/global.css`

**หน้าที่:** Import design-tokens แล้วเพิ่ม global styles อื่นๆ

```css
/* src/assets/global.css */
@import './design-tokens.css';  /* ← Import tokens เข้ามา */

*::-webkit-scrollbar {
  width: var(--scrollbar-size);
}
```

**ทำไมต้อง import ที่นี่?**
- เพื่อให้ tokens ถูกโหลดก่อน styles อื่นๆ
- Browser จะได้รู้จัก `--ssk-*` ก่อนใช้งาน

---

### 3️⃣ `main.ts` - Bundle CSS เข้า Library

**ตำแหน่ง:** `src/main.ts`

**หน้าที่:** Import CSS เพื่อให้ Vite bundle เข้า `dist/style.css`

```typescript
/* src/main.ts */
import "./assets/fonts.css";
import "./assets/global.css";  /* ← Vite จะ bundle ไฟล์นี้ */
```

**สิ่งที่ Vite ทำ:**
```bash
npm run build
# → Vite อ่าน main.ts
# → เจอ import "./assets/global.css"
# → อ่าน global.css
# → เจอ @import './design-tokens.css'
# → รวมทั้งหมดเข้า dist/style.css
```

---

### 4️⃣ `theme.ts` - TypeScript Types เท่านั้น! ⚠️

**ตำแหน่ง:** `src/types/theme.ts`

**หน้าที่:** ประกาศ TypeScript Types **ไม่ใช่** ประกาศ CSS Variables!

```typescript
/* src/types/theme.ts */
export type SpacingSize =
  | "none" | "xxs" | "xs" | "sm" | "md" | "lg" | "xl"
  | "2xl" | "3xl" | ... | "15xl";

export const cssVar = (...key: string[]) => {
  return `--ssk-${key.join('-')}`;
};
```

**สิ่งที่มีอยู่ใน theme.ts:**
- ✅ TypeScript Types (เช่น `SpacingSize`, `BorderRadiusSize`)
- ✅ Helper functions (เช่น `cssVar()`)
- ❌ ไม่มีการประกาศ CSS Variables

---

## 🔄 Flow การทำงานแบบละเอียด

### Step 1: Developer เขียนโค้ด

```typescript
/* src/elements/button/button.ts */
import { css } from 'lit';

export const buttonStyles = css`
  :host {
    padding: var(--ssk-spacing-md);  /* ← ใช้ CSS Variable */
    border-radius: var(--ssk-radius-lg);
  }
`;
```

### Step 2: Build Time

```bash
npm run build
```

**Vite ทำงาน:**

```
1. อ่าน src/main.ts
   ├── import "./assets/global.css"
   │   ├── @import './design-tokens.css'  ← โหลด tokens
   │   └── *::-webkit-scrollbar { ... }
   │
2. รวมทุกอย่างเข้า dist/style.css
3. อ่าน src/elements/button/button.ts
4. แปลงเป็น JavaScript bundle
```

**Output ใน `dist/style.css`:**

```css
/* dist/style.css (bundled) */
:root {
  --ssk-space-8: 0.5rem;
  --ssk-spacing-md: var(--ssk-space-6);
  --ssk-radius-lg: var(--ssk-space-12);
}

*::-webkit-scrollbar {
  width: var(--scrollbar-size);
}
```

### Step 3: Runtime (Browser)

```
User ฝัง Library เข้าเว็บ
  ↓
โหลด dist/style.css  ← โหลด CSS Variables
  ↓
Browser รู้จัก --ssk-spacing-md, --ssk-radius-lg
  ↓
Component ใช้ var(--ssk-spacing-md) ได้
```

---

## 🎯 สรุป: ไฟล์ไหนทำอะไร?

| ไฟล์ | ประเภท | หน้าที่ | Output |
|------|--------|---------|--------|
| **design-tokens.css** | CSS | ประกาศ `--ssk-*` variables | CSS Variables |
| **global.css** | CSS | รวม tokens + global styles | CSS Bundle |
| **main.ts** | TypeScript | Import CSS ให้ Vite bundle | Build instruction |
| **theme.ts** | TypeScript | Types + Helpers | IntelliSense |

---

## 💡 ทำไม theme.ts ไม่ประกาศ CSS Variables?

### CSS Variables อยู่ใน CSS ไม่ใช่ JavaScript!

```css
/* ✅ CSS - ถูกต้อง */
:root {
  --ssk-spacing-md: var(--ssk-space-6);
}
```

```typescript
/* ❌ TypeScript - ผิดวิธี! */
export const theme = {
  '--ssk-spacing-md': 'var(--ssk-space-6)'  // ไม่ใช้แบบนี้
};
```

### แล้ว theme.ts ใช้ทำอะไร?

**1. TypeScript Types - ให้ IDE แนะนำค่า**

```typescript
/* theme.ts */
export type SpacingSize = "none" | "xxs" | "xs" | ... | "15xl";
```

```typescript
/* ใน component */
interface Props {
  padding: SpacingSize;  /* ← IDE แนะนำค่าที่ถูกต้อง */
}

const props: Props = {
  padding: "md"  /* ✅ ถูกต้อง */
  // padding: "xdd"  /* ❌ TypeScript error */
};
```

**2. Helper Functions - สร้างชื่อ CSS Variable**

```typescript
/* theme.ts */
export const cssVar = (...key: string[]) => {
  return `--ssk-${key.join('-')}`;
};

/* ใน component */
const spacingVar = cssVar('spacing', 'md');  // "--ssk-spacing-md"
element.style.padding = `var(${spacingVar})`;
```

---

## 🔗 การเชื่อมโยงระหว่างไฟล์

### Diagram แบบเต็ม

```
┌─────────────────────────────────────────────────────────────┐
│                    DEVELOPMENT TIME                         │
└─────────────────────────────────────────────────────────────┘

┌──────────────────────┐
│ design-tokens.css    │  ← ประกาศ CSS Variables
│ :root {              │
│   --ssk-space-8: ... │
│   --ssk-spacing-md:  │
│     var(--ssk-...)  │
│ }                    │
└────────┬─────────────┘
         │ @import
┌────────▼─────────────┐
│ global.css           │  ← รวม CSS ทั้งหมด
│ @import tokens       │
│ + scrollbar styles   │
└────────┬─────────────┘
         │ import
┌────────▼─────────────┐
│ main.ts              │  ← Entry Point
│ import "./global.css"│
└────────┬─────────────┘
         │
┌────────▼─────────────┐
│ Vite Build           │  ← Bundle ทุกอย่าง
└────────┬─────────────┘
         │
┌────────▼─────────────┐
│ dist/style.css       │  ← Output (รวมทุกอย่าง)
└──────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      RUNTIME (Browser)                      │
└─────────────────────────────────────────────────────────────┘

dist/style.css  →  Browser รู้จัก --ssk-*  →  Components ใช้งานได้

┌──────────────────────┐
│ theme.ts             │  ← TypeScript Types เท่านั้น!
│ type SpacingSize     │     (ไม่ใช่ CSS Variables)
│ const cssVar()       │
└──────────────────────┘
         │
         │ ให้ IntelliSense และ Type Checking
         ▼
┌──────────────────────┐
│ Your Component       │  ← ใช้ Types + CSS Variables
│ props: SpacingSize   │
│ style: var(--ssk-...)│
└──────────────────────┘
```

---

## 🧪 ทดสอบดูว่าทำงานจริงไหม

### 1. Build แล้วดู output

```bash
npm run build
cat dist/style.css | grep "ssk-spacing"
```

ควรเห็น:
```css
--ssk-spacing-md: var(--ssk-space-6);
```

### 2. ตรวจสอบใน Browser DevTools

```javascript
// พิมพ์ใน Console
getComputedStyle(document.documentElement)
  .getPropertyValue('--ssk-spacing-md')
// ผลลัพธ์: "var(--ssk-space-6)" หรือ "0.375rem"
```

### 3. ใช้ใน Component

```css
/* ใน component ของคุณ */
.my-element {
  padding: var(--ssk-spacing-md);
}
```

เปิด DevTools → Elements → Computed → ดูค่า `padding`

---

## ❓ คำถามที่พบบ่อย

**Q: theme.ts ไม่ได้ประกาศ CSS Variables แล้วมันทำไม?**

A: `theme.ts` มีหน้าที่คนละอย่าง!
- **design-tokens.css** = ประกาศ CSS Variables (ให้ Browser ใช้)
- **theme.ts** = ประกาศ TypeScript Types (ให้ IDE/TypeScript ใช้)

**Q: ทำไมไม่รวมทุกอย่างใน theme.ts?**

A: CSS Variables ต้องอยู่ใน CSS ไม่ใช่ JavaScript!
- CSS Variables ใช้ใน `.css` files ได้เลย
- ถ้าอยู่ใน JavaScript ต้อง inject เข้า DOM (ซับซ้อนกว่า)

**Q: แล้วใครเชื่อม design-tokens.css เข้ากับ theme.ts?**

A: **ไม่ต้องเชื่อม!** เพราะ:
1. `design-tokens.css` → CSS Variables (สำหรับ Browser)
2. `theme.ts` → TypeScript Types (สำหรับ Developer)

พวกมัน "คู่กัน" แต่ไม่ได้ "เชื่อมกัน" โดยตรง

---

## 🎓 แนวคิดสำคัญ

### Separation of Concerns

```
CSS Variables    →  ค่าที่ Browser ใช้จริง (runtime)
TypeScript Types →  ค่าที่ Developer ใช้เขียนโค้ด (devtime)
```

### พวกมันต้อง match กัน

```css
/* design-tokens.css */
--ssk-spacing-md: var(--ssk-space-6);
```

```typescript
/* theme.ts */
export type SpacingSize = "none" | "xxs" | ... | "md" | ...;
```

ถ้าเพิ่ม token ใหม่ใน CSS ต้องเพิ่มใน Type ด้วย!

---

## 📝 Checklist เมื่อเพิ่ม Token ใหม่

1. ✅ เพิ่มใน `design-tokens.css`
   ```css
   --ssk-spacing-huge: var(--ssk-space-96);
   ```

2. ✅ เพิ่มใน `theme.ts`
   ```typescript
   export type SpacingSize = ... | "huge";
   ```

3. ✅ Rebuild
   ```bash
   npm run build
   ```

4. ✅ ใช้งานได้เลย!
   ```css
   .huge { padding: var(--ssk-spacing-huge); }
   ```
