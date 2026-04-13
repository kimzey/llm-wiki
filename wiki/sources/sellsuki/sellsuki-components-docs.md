---
title: "Sellsuki Components — Service Documentation"
type: source
source_file: raw/notes/central-component/docs.md
url: ""
published: 2026-03-01
tags: [web-components, lit, typescript, sellsuki, component-library, design-system, i18n, theme]
related: [wiki/sources/sellsuki-design-tokens-service.md, wiki/concepts/design-tokens.md, wiki/concepts/web-components-lit.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/central-component/docs.md|Original file]]

## สรุป

เอกสาร service documentation ของ `@sellsuki-org/sellsuki-components` — Web Components Library ที่ใช้ Lit Framework + TypeScript ครอบคลุม architecture, project structure, design system (tokens + colors + typography), 3 core systems (Theme/I18n/Toast), component catalog (30+), และ usage guide

## ประเด็นสำคัญ

- **Framework-agnostic Web Components**: ใช้ได้กับ React, Vue, Angular, Vanilla JS — เพราะสร้างด้วย Web Components Standard
- **Tech Stack**: Lit Framework + TypeScript + Vite build + Storybook documentation
- **3 Core Systems via Context Providers**:
  1. **Theme System**: CSS variables ผ่าน `ThemeProvider` — `parseThemeToCssVariables()` inject CSS vars เข้า Shadow DOM
  2. **I18n System**: `I18nProvider` + IndexedDB storage → offline support, multi-language, fallback language
  3. **Toast System**: `ToastProvider` + in-memory store + polling 100ms → auto-dismiss notifications
- **CSS Variable Fallback Chain**: component-specific → color palette → hardcoded default
- **Shadow DOM** scoping ป้องกัน style leak ออกนอก component
- **30+ components**: ครอบคลุม Form, Content, Feedback, Navigation, Data Display, Modal, Calendar, etc.

## ข้อมูล / หลักฐาน ที่น่าสนใจ

**Component Lifecycle ใน Lit:**
```
Definition → Registration → Instantiation → Property Setting → Context Injection → Render → Shadow DOM → Reactive Updates
```

**CSS Variable Fallback Pattern:**
```typescript
const color = parseVariables(
  cssVar("colors", "primary", 500),  // 1. Component-specific
  cssVar("colors", "primary"),        // 2. Color palette
  "#44aef8"                           // 3. Default value
);
```

**Design System ขยายใหญ่กว่า spacing** — มี Color Palette (60+ colors), Typography, Animation tokens ด้วย

**Package:** `@sellsuki-org/sellsuki-components` v0.1.2

**Installation:**
```bash
npm install @sellsuki-org/sellsuki-components
```

**Provider Pattern ที่ต้อง wrap:**
```html
<ssk-theme-provider>
  <ssk-i18n-provider lang="th">
    <ssk-toast-provider>
      <!-- App content -->
    </ssk-toast-provider>
  </ssk-i18n-provider>
</ssk-theme-provider>
```

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่มีข้อขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/design-tokens|Design Tokens]]
- [[wiki/concepts/web-components-lit|Web Components & Lit Framework]]
