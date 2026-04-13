# 📘 Design Tokens Service - Complete Documentation

> **Sellsuki Components Library** | Version 1.0.0 | March 2026

---

## 📚 Table of Contents

1. [Executive Summary](#executive-summary)
2. [System Architecture](#system-architecture)
3. [File-by-File Deep Dive](#file-by-file-deep-dive)
4. [Data Flow - End to End](#data-flow---end-to-end)
5. [How Everything Connects](#how-everything-connects)
6. [Real-World Examples](#real-world-examples)
7. [Troubleshooting Guide](#troubleshooting-guide)
8. [Quick Reference](#quick-reference)

---

## Executive Summary

### What is Design Tokens Service?

**Design Tokens Service** คือระบบจัดการค่าตัวแปรการออกแบบ (Design Variables) ของ Sellsuki Component Library ที่ทำให้:
- Developer ใช้ค่า spacing/radius/border ได้อย่างสม่ำเสมอ
- Design เปลี่ยนค่าได้จากจุดเดียว กระทบทั้งระบบ
- มี TypeScript Support แบบเต็มรูปแบบ
- Scale ได้ง่ายเมื่อ design system เติบโต

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   DEVELOPMENT PHASE                         │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  CSS Files (Design Tokens)     TypeScript Files (Types)     │
│  ┌──────────────────────┐     ┌──────────────────────┐      │
│  │ design-tokens.css    │     │ theme.ts             │      │
│  │ (ประกาศ --ssk-*)     │     │ (ประกาศ Types)        │      │
│  └──────────┬───────────┘     └──────────────────────┘      │
│             │                                               │
│  ┌──────────▼───────────┐                                   │
│  │ global.css           │                                   │
│  │ (import tokens)      │                                   │
│  └──────────┬───────────┘                                   │
│             │                                               │
│  ┌──────────▼───────────┐     ┌──────────────────────┐      │
│  │ main.ts              │────│ Type Checking         │      │
│  │ (import CSS)         │     │ (IntelliSense)        │      │
│  └──────────┬───────────┘     └──────────────────────┘      │
│             │                                               │
└─────────────┼───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│                      BUILD PHASE                            │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Vite bundles CSS → dist/style.css                           │
│  TypeScript compiles → dist/*.js                             │
│                                                               │
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│                      RUNTIME PHASE                          │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Browser loads:                                              │
│  • dist/style.css → CSS Variables available globally          │
│  • dist/*.js → Components use var(--ssk-*)                   │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## System Architecture

### The Two Parallel Systems ⚡

Design Tokens Service ทำงานด้วย 2 ระบบที่ควบคู่กัน (แต่ไม่ได้เชื่อมต่อโดยตรง):

```
┌─────────────────────────────────────────────────────────────┐
│              SYSTEM 1: CSS VARIABLES (Runtime)               │
│              ให้ Browser ใช้จริงขณะทำงาน                       │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  design-tokens.css → global.css → main.ts                   │
│       (ประกาศ)         (รวม)        (bundle)                 │
│         │                                         │           │
│         ▼                                         ▼           │
│  Browser knows --ssk-spacing-md                    │           │
│  Browser knows --ssk-radius-lg                     │           │
│                                                       │       │
└───────────────────────────────────────────────────┼───────┘
                                                    │
┌───────────────────────────────────────────────────▼───────┐
│           SYSTEM 2: TYPESCRIPT TYPES (Devtime)              │
│           ให้ Developer ใช้เวลาเขียนโค้ด                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  theme.ts                                                     │
│  ├── export type SpacingSize                                 │
│  ├── export type BorderRadiusSize                            │
│  └── export const cssVar()                                   │
│         │                                                     │
│         ▼                                                     │
│  IDE knows: padding="md" is valid                            │
│  TypeScript knows: padding="xdd" is ERROR                    │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Why Two Systems?

| Phase | System | Purpose | Example |
|-------|--------|---------|---------|
| **Development** | TypeScript Types | Prevent errors, IntelliSense | IDE warns if you type `"xdd"` |
| **Runtime** | CSS Variables | Apply styles in browser | `var(--ssk-spacing-md)` becomes `6px` |

---

## File-by-File Deep Dive

### 📄 1. `src/assets/design-tokens.css`

**หน้าที่:** ประกาศ CSS Custom Properties ทั้งหมด (Single Source of Truth)

```css
/* src/assets/design-tokens.css */
:root {
  /* Tier 1: Primitives */
  --ssk-space-8: 0.5rem;  /* 8px */

  /* Tier 2: Semantic */
  --ssk-spacing-md: var(--ssk-space-6);  /* 6px */
  --ssk-radius-lg: var(--ssk-space-12);  /* 12px */
  --ssk-border-sm: var(--ssk-space-4);   /* 4px */
}
```

**สิ่งที่มีอยู่:**
- ✅ CSS Variables ทั้งหมด (spacing, radius, border)
- ✅ 2-Tier Architecture (Primitives → Semantic)
- ❌ ไม่มี JavaScript/TypeScript

**เมื่อไหร่ที่ Browser อ่านไฟล์นี้?**
- ผ่าน `global.css` → `main.ts` → `dist/style.css`
- Browser โหลด `dist/style.css` แล้วรู้จักทุก `--ssk-*`

**ตัวอย่างการใช้ใน CSS:**
```css
.my-button {
  padding: var(--ssk-spacing-md);      /* 6px */
  border-radius: var(--ssk-radius-lg); /* 12px */
}
```

---

### 📄 2. `src/assets/global.css`

**หน้าที่:** รวม CSS ทั้งหมดเข้าด้วยกัน (Entry Point สำหรับ CSS)

```css
/* src/assets/global.css */
@import './design-tokens.css';  /* ← โหลด tokens ก่อน */

*::-webkit-scrollbar {
  width: var(--scrollbar-size);
}
```

**ทำไมต้อง import tokens ก่อน?**
```css
/* ❌ WRONG - tokens ยังไม่ถูกประกาศ */
*::-webkit-scrollbar {
  width: var(--ssk-spacing-md);  /* undefined! */
}

@import './design-tokens.css';

/* ✅ CORRECT - tokens ถูกประกาศแล้ว */
@import './design-tokens.css';

.my-component {
  padding: var(--ssk-spacing-md);  /* works! */
}
```

**สิ่งที่มีอยู่:**
- ✅ Import `design-tokens.css`
- ✅ Global styles (scrollbar, resets)
- ✅ เป็นจุดรวม CSS ทั้งหมด

---

### 📄 3. `src/main.ts`

**หน้าที่:** Entry Point ของ Library - บอก Vite ว่าต้อง bundle อะไรบ้าง

```typescript
/* src/main.ts */
export * from "./types/theme";         // Export types
export * from "./elements/button";     // Export components

import "./assets/fonts.css";           // บอก Vite: bundle เอาไปด้วย
import "./assets/global.css";          // บอก Vite: bundle เอาไปด้วย
```

**สิ่งที่ Vite ทำเมื่อเจอ `import "./assets/global.css"`:**

```
1. Vite อ่าน main.ts
2. เจอ import "./assets/global.css"
3. ตามไปอ่าน global.css
4. เจอ @import './design-tokens.css'
5. ตามไปอ่าน design-tokens.css
6. รวมทั้งหมดเข้า dist/style.css
```

**Output ใน `dist/style.css`:**
```css
/* dist/style.css - รวมทุกอย่างแล้ว */
:root {
  --ssk-space-8: 0.5rem;
  --ssk-spacing-md: var(--ssk-space-6);
  --ssk-radius-lg: var(--ssk-space-12);
}

*::-webkit-scrollbar {
  width: var(--scrollbar-size);
}
```

---

### 📄 4. `src/types/theme.ts`

**หน้าที่:** TypeScript Types และ Helper Functions (ไม่ใช่ CSS!)

```typescript
/* src/types/theme.ts */

// Types - ให้ TypeScript รู้จักค่าที่ใช้ได้
export type SpacingSize =
  | "none" | "xxs" | "xs" | "sm" | "md" | "lg" | "xl"
  | "2xl" | "3xl" | ... | "15xl";

export type BorderRadiusSize =
  | "none" | "xxs" | "xs" | "sm" | "md" | "lg" | "xl"
  | "2xl" | "3xl" | "4xl" | "full";

export type BorderWidthSize = "none" | "xxs" | "xs" | "sm";

// Helper Functions - ช่วยสร้างชื่อ CSS Variable
export const cssVar = (...key: string[]) => {
  return `--ssk-${key.join('-')}`;
};
```

**สิ่งที่มีอยู่:**
- ✅ TypeScript Types (สำหรับ IntelliSense, Type Checking)
- ✅ Helper Functions (สำหรับ generate CSS variable names)
- ❌ ไม่มี CSS Variables (ไม่ใช่หน้าที่ของไฟล์นี้)

**ตัวอย่างการใช้:**

```typescript
import type { SpacingSize, BorderRadiusSize } from '@/types/theme';
import { cssVar } from '@/types/theme';

// Type-safe props
interface ButtonProps {
  padding?: SpacingSize;          // ← IDE แนะนำค่า
  borderRadius?: BorderRadiusSize; // ← TypeScript เช็ค
}

// ✅ CORRECT
const props: ButtonProps = {
  padding: "md",           // Valid
  borderRadius: "lg"       // Valid
};

// ❌ ERROR
const props2: ButtonProps = {
  padding: "xdd",          // TypeScript error!
  borderRadius: "wrong"    // TypeScript error!
};

// Helper function
const spacingVar = cssVar('spacing', 'md'); // "--ssk-spacing-md"
element.style.padding = `var(${spacingVar})`;
```

---

### 📄 5. `.storybook/preview-head.html`

**หน้าที่:** Inject CSS เข้า Storybook Preview

```html
<!-- .storybook/preview-head.html -->
<link rel="stylesheet" href="/assets/fonts.css" />
<link rel="stylesheet" href="/assets/global.css" />
```

**ทำไมต้องมี?**
- Storybook เป็นเว็บแยก ต้องโหลด CSS เอง
- ไม่ได้โหลดจาก `main.ts` เหมือน production

**Flow:**
```
Storybook Start
  ↓
Load preview-head.html
  ↓
Load /assets/global.css
  ↓
Design tokens available in Storybook
  ↓
Components display correctly
```

---

## Data Flow - End to End

### Phase 1: Development Time

```
┌─────────────────────────────────────────────────────────────┐
│ Developer writes code                                        │
└─────────────────────────────────────────────────────────────┘

Component File (e.g., button.ts)
├── Import types from theme.ts
│   └── type SpacingSize = "md" | "lg" | ...
│
└── Use CSS variables in styles
    └── padding: var(--ssk-spacing-md)

Simultaneously:
├── TypeScript checks types (devtime)
│   └── Is "md" valid? Yes ✅
│
└── CSS variables defined (to be used at runtime)
    └── --ssk-spacing-md exists in design-tokens.css
```

### Phase 2: Build Time

```bash
$ npm run build
```

```
┌─────────────────────────────────────────────────────────────┐
│ Vite Build Process                                          │
└─────────────────────────────────────────────────────────────┘

Step 1: Parse Entry Points
├── src/main.ts
│   ├── export * from "./types/theme"        → Generate .d.ts
│   ├── export * from "./elements/button"    → Compile to .js
│   └── import "./assets/global.css"         → Bundle CSS
│
Step 2: Process CSS
├── src/assets/global.css
│   ├── @import './design-tokens.css'
│   │   └── :root { --ssk-spacing-md: ... }
│   └── *::-webkit-scrollbar { ... }
│
Step 3: Bundle Everything
├── dist/style.css                           ← All CSS merged
├── dist/index.js                            ← All components
├── dist/types/*.d.ts                        ← TypeScript definitions
└── dist/sellsuki-components.umd.cjs         ← UMD bundle
```

### Phase 3: Runtime (Browser)

```
┌─────────────────────────────────────────────────────────────┐
│ User's Application loads the library                        │
└─────────────────────────────────────────────────────────────┘

index.html
├── <link href="dist/style.css">      ← Load CSS first
└── <script src="dist/index.js">      ← Then JavaScript

Browser Rendering:
1. Parse dist/style.css
   ├── :root {
   │     --ssk-spacing-md: var(--ssk-space-6);
   │     --ssk-space-6: 0.375rem;  ← 6px
   │   }
   │
2. Render Components
   └── <ssk-button style="padding: var(--ssk-spacing-md)">
       └── Browser computes: padding: 6px
```

---

## How Everything Connects

### The Big Picture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                        DEVELOPMENT                               │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────┐          ┌──────────────────────┐
│ design-tokens.css    │          │ theme.ts             │
│                      │          │                      │
│ :root {              │          │ export type          │
│   --ssk-space-8: ... │          │   SpacingSize = ...  │
│   --ssk-spacing-md:  │          │                      │
│     var(--ssk-...)  │          │ export const cssVar() │
│ }                    │          └──────────────────────┘
└──────────┬───────────┘                    │
           │                               │
           │ คู่กัน (Parallel)              │ คู่กัน (Parallel)
           │ แต่ไม่เชื่อมตรง              │ แต่ไม่เชื่อมตรง
           │                               │
           ▼                               ▼
┌──────────────────────────────────────────────────────┐
│ global.css                                           │
│ ┌─────────────────────────────────────────────┐      │
│ │ @import './design-tokens.css'                │      │
│ │ *::-webkit-scrollbar { ... }                 │      │
│ └─────────────────────────────────────────────┘      │
└────────┬─────────────────────────────────────────────┘
         │ import
┌────────▼─────────────────────────────────────────────┐
│ main.ts                                              │
│ ┌─────────────────────────────────────────────┐      │
│ │ import "./assets/global.css"                 │ ← CSS Bundle
│ │ export * from "./types/theme"                │ ← TS Export
│ │ export * from "./elements/button"            │ ← Components
│ └─────────────────────────────────────────────┘      │
└────────┬─────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│                      BUILD                              │
│           (Vite bundles everything)                     │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│                      OUTPUT                             │
├─────────────────────────────────────────────────────────┤
│ dist/                                                  │
│ ├── style.css          ← All CSS (tokens + styles)     │
│ ├── index.js           ← All components                │
│ ├── types/theme.d.ts   ← TypeScript definitions        │
│ └── sellsuki-components.umd.cjs                       │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│                    USER'S APP                           │
│              (Runtime - Browser)                        │
├─────────────────────────────────────────────────────────┤
│ 1. Load dist/style.css                                 │
│    → Browser knows --ssk-spacing-md                     │
│                                                         │
│ 2. Load dist/index.js                                  │
│    → Components render                                  │
│    → Use var(--ssk-spacing-md)                         │
│    → Browser computes: 6px                             │
└─────────────────────────────────────────────────────────┘
```

### Key Insight: Two Separate Connections

```
Connection 1: CSS Flow (for Browser)
design-tokens.css → global.css → main.ts → dist/style.css → Browser

Connection 2: TypeScript Flow (for Developer)
theme.ts → main.ts (export) → dist/types/theme.d.ts → IDE/Developer

These flows DON'T connect directly!
They work in parallel but serve different purposes.
```

---

## Real-World Examples

### Example 1: Creating a Button Component

```typescript
/* src/elements/button/button.ts */
import { css } from 'lit';
import type { SpacingSize, BorderRadiusSize } from '@/types/theme';

// TypeScript Interface (uses types from theme.ts)
interface ButtonProps {
  padding?: SpacingSize;
  borderRadius?: BorderRadiusSize;
}

// Component Styles (uses CSS variables from design-tokens.css)
export const buttonStyles = css`
  :host {
    /* These CSS variables come from design-tokens.css */
    padding: var(--ssk-spacing-md);
    border-radius: var(--ssk-radius-md);
    border-width: var(--ssk-border-xs);
  }
`;

// Component Logic (uses TypeScript types)
export class Button extends LitElement {
  @property({ type: String })
  padding: SpacingSize = 'md';  // ← Type-checked!

  @property({ type: String })
  borderRadius: BorderRadiusSize = 'md';  // ← Type-checked!
}
```

**What happens where:**

| Part | System | Source |
|------|--------|--------|
| `padding: SpacingSize` | TypeScript | `theme.ts` |
| `padding: var(--ssk-spacing-md)` | CSS | `design-tokens.css` |
| IntelliSense showing "md", "lg" | TypeScript | `theme.ts` |
| Browser applying 6px padding | CSS | `design-tokens.css` |

---

### Example 2: Dynamic Spacing with Helper Function

```typescript
import { cssVar } from '@/types/theme';

// Generate CSS variable name
function getSpacing(size: 'sm' | 'md' | 'lg') {
  return cssVar('spacing', size);
}

// Usage
const spacingVar = getSpacing('md');  // "--ssk-spacing-md"

// Apply to element
element.style.setProperty('padding', `var(${spacingVar})`);
// Result: padding: var(--ssk-spacing-md) → 6px
```

---

### Example 3: Complete Flow in Action

```typescript
/* ===== DEVELOPMENT TIME ===== */

// 1. Developer writes component
import type { SpacingSize } from './types/theme';

interface CardProps {
  padding: SpacingSize;  // TypeScript checks this
}

const cardStyles = css`
  :host {
    padding: var(--ssk-spacing-lg);  // CSS provides this
  }
`;

/* ===== BUILD TIME ===== */

// 2. Developer runs: npm run build

// Vite does:
// - Reads src/main.ts
// - Finds import "./assets/global.css"
// - Reads global.css
// - Finds @import './design-tokens.css'
// - Reads design-tokens.css
// - Bundles everything into dist/style.css

/* ===== RUNTIME ===== */

// 3. User's app loads library

// index.html:
// <link rel="stylesheet" href="node_modules/@sellsuki/components/dist/style.css">
// <script src="node_modules/@sellsuki/components/dist/index.js"></script>

// Browser:
// 1. Parses dist/style.css
//    → Knows --ssk-spacing-lg = var(--ssk-space-8) = 0.5rem = 8px
// 2. Executes dist/index.js
//    → Renders <ssk-card>
//    → Applies padding: var(--ssk-spacing-lg)
//    → Computes: padding: 8px
```

---

## Troubleshooting Guide

### Problem 1: CSS Variable Not Working

**Symptom:**
```css
.my-component {
  padding: var(--ssk-spacing-md);
}
/* Result: padding has no effect */
```

**Diagnosis:**
```bash
# 1. Check if CSS is loaded
grep "ssk-spacing-md" dist/style.css

# Should see:
# --ssk-spacing-md: var(--ssk-space-6);
```

**Solutions:**

| Issue | Solution |
|-------|----------|
| CSS not imported | Add `import "./assets/global.css"` to `main.ts` |
| Wrong variable name | Check `design-tokens.css` for exact name |
| Browser cache | Hard refresh: Ctrl+Shift+R |

---

### Problem 2: TypeScript Error

**Symptom:**
```typescript
const props: SpacingSize = 'md';
// Error: Type '"md"' is not assignable to type 'SpacingSize'
```

**Diagnosis:**
```typescript
// Check if type is exported
import type { SpacingSize } from '@/types/theme';

// Check if value exists in type
type SpacingSize = "none" | "xxs" | ... | "15xl";  // Is "md" here?
```

**Solutions:**

| Issue | Solution |
|-------|----------|
| Type not exported | Add `export type SpacingSize` to `theme.ts` |
| Value not in type | Add value to type union |
| Import path wrong | Check import path is correct |

---

### Problem 3: Storybook Shows Wrong Styles

**Symptom:** Components look different in Storybook vs production

**Diagnosis:**
```html
<!-- Check .storybook/preview-head.html -->
<link rel="stylesheet" href="/assets/global.css" />
```

**Solutions:**

| Issue | Solution |
|-------|----------|
| CSS not loaded | Add `<link>` to `preview-head.html` |
| Wrong path | Check Storybook static assets config |
| Cache | Restart Storybook: `npm run storybook` |

---

## Quick Reference

### File Responsibilities at a Glance

| File | Type | Responsibility | Output |
|------|------|-----------------|--------|
| **design-tokens.css** | CSS | Define `--ssk-*` variables | CSS Variables |
| **global.css** | CSS | Import tokens + global styles | CSS Bundle |
| **main.ts** | TypeScript | Export all + import CSS | Build entry |
| **theme.ts** | TypeScript | Types + helpers | IntelliSense |
| **preview-head.html** | HTML | Load CSS in Storybook | Storybook styles |

### Token Naming Convention

```
--ssk-{category}-{size}

Category Options:
  - spacing (for padding, margin, gap)
  - radius (for border-radius)
  - border (for border-width)
  - space (primitives only - Tier 1)

Size Options:
  - none, xxs, xs, sm, md, lg, xl
  - 2xl, 3xl, 4xl, 5xl, 6xl, 7xl, 8xl, 9xl
  - 10xl, 11xl, 12xl, 13xl, 14xl, 15xl
  - full (for radius only)
```

### Common Tasks

| Task | File(s) to Edit |
|------|-----------------|
| Add new spacing token | `design-tokens.css` + `theme.ts` |
| Change token value | `design-tokens.css` only |
| Add new component using tokens | Component file (use existing tokens) |
| Fix TypeScript error | Check `theme.ts` types |
| Debug CSS variable | Check `dist/style.css` output |

---

## Summary

### The Golden Rules

1. **CSS Variables live in CSS files** (`design-tokens.css`)
   - Never put CSS variables in TypeScript
   - They're for browser runtime, not developer time

2. **TypeScript Types live in TS files** (`theme.ts`)
   - Never put types in CSS
   - They're for developer tooling, not runtime

3. **The two systems work in parallel, not together**
   - CSS variables → what browser uses
   - TypeScript types → what developers use
   - They must match, but don't directly connect

4. **Main flow is simple**
   ```
   design-tokens.css → global.css → main.ts → dist/style.css → Browser
   ```

5. **When adding new tokens**
   - Add CSS variable to `design-tokens.css`
   - Add type to `theme.ts`
   - Rebuild: `npm run build`

---

**© 2026 Sellsuki Design System. All rights reserved.**

For questions or issues, please refer to:
- `docs/design-tokens.md` - Usage documentation
- `docs/design-tokens-flow.md` - Flow diagrams
- Or contact the Design System team
