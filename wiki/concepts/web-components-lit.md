---
title: "Web Components & Lit Framework"
type: concept
tags: [web-components, lit, shadow-dom, custom-elements, typescript, framework-agnostic]
sources: [docs.md]
related: [wiki/concepts/design-tokens.md, wiki/sources/sellsuki-components-docs.md]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

**Web Components** คือ Web Standard สำหรับสร้าง custom HTML elements ที่ใช้ได้ทุก framework — **Lit** คือ library ขนาดเล็กที่ทำให้เขียน Web Components ได้ง่ายขึ้น ผ่าน decorators, reactive properties, และ tagged template literals สำหรับ HTML/CSS

## อธิบาย

Web Components ประกอบด้วย 3 Web API:
1. **Custom Elements**: `customElements.define('my-button', MyButton)` — register HTML tag ใหม่
2. **Shadow DOM**: สร้าง encapsulated DOM tree — styles ไม่ leak ออกนอก หรือเข้ามาใน
3. **HTML Templates**: `<template>` element สำหรับ reusable markup

**Lit Framework** wrap Web Components APIs ให้ใช้งานง่ายขึ้น:
```typescript
@customElement("ssk-button")
export class Button extends LitElement {
  @property({ type: String })
  variant: "solid" | "outline" | "ghost" = "solid";

  render() {
    return html`<button class="${this.variant}"><slot></slot></button>`;
  }

  static styles = css`
    :host { display: inline-block; }
    button { padding: var(--ssk-spacing-md); }
  `;
}
```

### Context System ใน Sellsuki Components

Lit's context protocol ให้ parent component share data ลงไปยัง descendants โดยไม่ต้องส่ง props ทีละ level:

```typescript
// Provider
@provide({ context: themeContext })
theme: Theme = defaultTheme;

// Consumer (ใน any descendant)
@consume({ context: themeContext })
theme?: Theme;
```

ใช้สำหรับ Theme, I18n, และ Toast systems

### Component Lifecycle

```
@customElement → Registration → Instantiation → Property Setting
→ Context Injection → render() → Static Styles → Shadow DOM Attachment
→ Reactive Updates (loop)
```

## ประเด็นสำคัญ

- **Framework-agnostic**: Web Components เป็น browser standard — React, Vue, Angular, Vanilla JS ใช้ได้หมด
- **Shadow DOM isolation**: CSS ไม่ leak ออก/เข้า — styles ถูก scope ใน component เสมอ
- **Reactive properties**: `@property()` decorator ทำให้ component re-render อัตโนมัติเมื่อ prop เปลี่ยน
- **Slots**: `<slot>` สำหรับ content projection — named slots สำหรับ prefix/postfix patterns
- **Performance**: Lit ตัวเล็กมาก (~5kb), re-render เฉพาะ parts ที่เปลี่ยน

## ตัวอย่าง / กรณีศึกษา

**Sellsuki Component Pattern:**
```typescript
@customElement("ssk-my-component")
export class MyComponent extends LitElement {
  @consume({ context: themeContext })
  theme?: Theme;

  @property({ type: String })
  size: Size = "md";

  render() {
    return html`
      ${parseThemeToCssVariables(this.theme?.components?.myComponent)}
      <div class="my-component"><slot></slot></div>
    `;
  }

  static styles = css`
    .my-component { padding: var(--ssk-padding); }
  `;
}
```

**CSS Variable Fallback Chain:**
```typescript
const color = parseVariables(
  cssVar("colors", "primary", 500),  // 1. component-specific override
  cssVar("colors", "primary"),        // 2. palette default
  "#44aef8"                           // 3. hardcoded fallback
);
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/design-tokens|Design Tokens]] — CSS variables จาก design tokens ใช้ใน Shadow DOM ของ Web Components ผ่าน `var(--ssk-*)`

## แหล่งที่มา

- [[wiki/sources/sellsuki/sellsuki-components-docs|Sellsuki Components — Service Documentation]]
