---
title: "Sellsuki Design Tokens Service — Complete Documentation"
type: source
source_file: raw/notes/central-component/DESIGN_TOKENS_SERVICE.md
url: ""
published: 2026-03-01
tags: [design-tokens, css-variables, typescript, sellsuki, component-library]
related: [wiki/sources/sellsuki-design-tokens-flow.md, wiki/sources/sellsuki-design-tokens-reference.md, wiki/concepts/design-tokens.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/central-component/DESIGN_TOKENS_SERVICE.md|Original file]]

## สรุป

เอกสาร complete documentation ของ Design Tokens Service ใน Sellsuki Component Library อธิบายสถาปัตยกรรม 2 ระบบที่ทำงานคู่กัน (CSS Variables + TypeScript Types), data flow ทุก phase (dev/build/runtime), troubleshooting guide และ real-world examples

## ประเด็นสำคัญ

- **2 ระบบคู่ขนาน**: CSS Variables (runtime, สำหรับ browser) + TypeScript Types (devtime, สำหรับ developer) ทำงานคู่กันแต่ไม่ได้เชื่อมตรง
- **Single Source of Truth**: `design-tokens.css` คือที่เดียวที่ประกาศ CSS variables ทั้งหมด (`--ssk-*`)
- **Import chain**: `design-tokens.css` → `global.css` → `main.ts` → `dist/style.css` → Browser
- **`theme.ts` ไม่ได้ประกาศ CSS Variables** — มีหน้าที่แค่ TypeScript Types + helper function `cssVar()`
- **Vite bundling**: เมื่อ build, Vite ตาม import chain รวม CSS ทั้งหมดเป็น `dist/style.css` เดียว
- **Storybook ต้องโหลด CSS เอง** ผ่าน `.storybook/preview-head.html` เพราะไม่ได้โหลดจาก `main.ts`

## ข้อมูล / หลักฐาน ที่น่าสนใจ

**Token naming convention:**
```
--ssk-{category}-{size}
Category: spacing | radius | border | space (primitives)
```

**ไฟล์และหน้าที่:**
| ไฟล์ | ประเภท | หน้าที่ |
|------|--------|---------|
| `design-tokens.css` | CSS | ประกาศ `--ssk-*` (Single Source of Truth) |
| `global.css` | CSS | รวม tokens + global styles |
| `main.ts` | TypeScript | Entry point: import CSS → Vite bundles |
| `theme.ts` | TypeScript | Types + `cssVar()` helper เท่านั้น |
| `.storybook/preview-head.html` | HTML | โหลด CSS ใน Storybook |

**Golden Rules:**
1. CSS Variables อยู่ใน CSS files เท่านั้น
2. TypeScript Types อยู่ใน TS files เท่านั้น
3. 2 ระบบทำงานคู่กัน ไม่เชื่อมตรง — ต้อง match กันด้วยมือ

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่มีข้อขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/design-tokens|Design Tokens]]
