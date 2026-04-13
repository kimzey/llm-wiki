---
title: "Design Tokens"
type: concept
tags: [design-tokens, css-variables, design-system, frontend, component-library]
sources: [DESIGN_TOKENS_SERVICE.md, design-tokens-flow.md, design-tokens.md, docs.md]
related: [wiki/concepts/web-components-lit.md, wiki/sources/sellsuki-design-tokens-service.md, wiki/sources/sellsuki-design-tokens-reference.md]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

**Design Tokens** คือค่าที่เก็บ "design decisions" ในรูปแบบตัวแปรที่ใช้ได้ทั้งใน CSS และ JavaScript — เป็น Single Source of Truth ของค่าที่ใช้ซ้ำในระบบ UI เช่น spacing, color, border-radius, typography ทำให้แก้ค่าจากที่เดียวกระทบทั้งระบบ

## อธิบาย

Design Tokens คือแนวคิดที่แยก "ค่าการออกแบบ" ออกจาก component code — แทนที่จะเขียน `padding: 8px` ตรงๆ ใช้ `padding: var(--ssk-spacing-lg)` แทน การเปลี่ยนขนาด spacing ทั้งระบบทำได้โดยแก้ค่า token เดียว

### Implementation ใน Sellsuki Components

ใน Sellsuki Components Library, Design Tokens ทำงานผ่าน 2 ระบบคู่ขนาน:

**System 1: CSS Variables (Runtime)**
สำหรับ browser ใช้จริงขณะทำงาน
```css
/* design-tokens.css */
:root {
  --ssk-spacing-md: var(--ssk-space-6);  /* 6px */
  --ssk-radius-lg: var(--ssk-space-12);  /* 12px */
}
```

**System 2: TypeScript Types (Devtime)**
สำหรับ developer ใช้เวลาเขียนโค้ด
```typescript
// theme.ts
export type SpacingSize = "none" | "xxs" | "xs" | "sm" | "md" | "lg" | ...;
export const cssVar = (...key: string[]) => `--ssk-${key.join('-')}`;
```

สองระบบนี้ทำงานคู่กันแต่ไม่เชื่อมตรง — developer ต้อง keep ให้ sync กันเอง

### 2-Tier Architecture

```
Tier 2: Semantic Tokens (ใช้ใน code)
  --ssk-spacing-md → var(--ssk-space-6)
  --ssk-radius-lg  → var(--ssk-space-12)
        ↓ var() reference
Tier 1: Primitives (ไม่ใช้โดยตรง)
  --ssk-space-6  → 0.375rem (6px)
  --ssk-space-12 → 0.75rem (12px)
```

เหตุผลที่มี 2 tiers:
- เปลี่ยน `md` spacing ทั้งระบบ → แก้แค่ `--ssk-spacing-md` ที่เดียว
- Scale ขนาดทั้งหมด → แก้ primitive values

### Build & Deploy Flow

```
design-tokens.css → global.css → main.ts → dist/style.css → Browser
```

Vite ตาม import chain รวม CSS ทั้งหมดเป็น `dist/style.css` เดียว browser โหลดครั้งเดียวรู้จักทุก `--ssk-*`

## ประเด็นสำคัญ

- **Single Source of Truth**: CSS variables ทั้งหมดอยู่ใน `design-tokens.css` ที่เดียว
- **Separation of concerns**: CSS สำหรับ runtime values, TypeScript สำหรับ type safety — ไม่ผสมกัน
- **Semantic > Primitive**: ใช้ `--ssk-spacing-md` ไม่ใช่ `--ssk-space-6` ใน component code
- **Manual sync required**: เพิ่ม CSS token ต้องเพิ่ม TS type ด้วยเสมอ ไม่มี auto-generate
- **2-tier benefits**: เปลี่ยนค่าจากจุดเดียว ไม่ต้อง search & replace ทั้ง codebase

## ตัวอย่าง / กรณีศึกษา

```css
/* ✅ ถูกต้อง — ใช้ Semantic Token */
.button {
  padding: var(--ssk-spacing-sm) var(--ssk-spacing-md);
  border-radius: var(--ssk-radius-md);
}

/* ❌ ผิด — hardcoded pixel */
.button {
  padding: 4px 6px;
  border-radius: 8px;
}
```

**Naming Convention:**
```
--ssk-{category}-{size}
category: spacing | radius | border | space (primitives)
size: none | xxs | xs | sm | md | lg | xl | 2xl | ... | 15xl | full
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/web-components-lit|Web Components & Lit Framework]] — design tokens ใช้เป็น CSS variables ใน Shadow DOM ของ Web Components
- Design tokens เป็นส่วนหนึ่งของ Design System ที่ใหญ่กว่า (colors, typography, animation ก็เป็น token)

## แหล่งที่มา

- [[wiki/sources/sellsuki-design-tokens-service|Sellsuki Design Tokens Service — Complete Documentation]]
- [[wiki/sources/sellsuki-design-tokens-flow|Sellsuki Design Tokens — Flow การทำงาน]]
- [[wiki/sources/sellsuki-design-tokens-reference|Sellsuki Design Tokens — Spacing System Reference]]
- [[wiki/sources/sellsuki-components-docs|Sellsuki Components — Service Documentation]]
