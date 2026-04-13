# Sellsuki Design Tokens - Spacing System

> **Official Documentation** | Version: 1.0.0 | Last Updated: March 2026

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Architecture - 2-Tier System](#architecture---2-tier-system)
3. [Token Reference](#token-reference)
4. [Usage Guide](#usage-guide)
5. [TypeScript Integration](#typescript-integration)
6. [Best Practices](#best-practices)
7. [Migration Guide](#migration-guide)
8. [FAQ](#faq)
9. [File Structure](#file-structure)

---

## Overview

### What are Design Tokens?

**Design Tokens** คือค่าที่เก็บข้อมูลการออกแบบ (design decisions) ในรูปแบบตัวแปร ที่สามารถนำไปใช้ได้ทั้งใน CSS และ JavaScript ช่วยให้:
- ✅ รักษาความสม่ำเสมอของ UI
- ✅ แก้ไขค่าได้จากจุดเดียว (Single Source of Truth)
- ✅ สเกลดีไซน์ได้ง่ายขึ้น
- ✅ ลดความผิดพลาดจากการ hardcoded values

### What's Included?

This implementation includes:

| Category | Tokens | Purpose |
|----------|--------|---------|
| **Spacing** | 16 levels (none → 15xl) | padding, margin, gap |
| **Border Radius** | 11 levels (none → full) | border-radius |
| **Border Width** | 4 levels (none → sm) | border-width |

---

## Architecture - 2-Tier System

### Why 2 Tiers?

```
┌─────────────────────────────────────────────────────────────┐
│                    TIER 2: SEMANTIC                         │
│              (What you USE in code)                         │
│  --ssk-spacing-md, --ssk-radius-lg, --ssk-border-sm         │
└────────────────────┬────────────────────────────────────────┘
                     │ var() reference
┌────────────────────▼────────────────────────────────────────┐
│                    TIER 1: PRIMITIVES                        │
│            (Base values - RARELY change)                    │
│   --ssk-space-0, --ssk-space-8, --ssk-space-16 ...          │
└─────────────────────────────────────────────────────────────┘
```

### Tier 1: Primitives (วัตถุดิบ)

ค่าตัวเลขพื้นฐานในหน่วย `rem` ที่ **ไม่ควรเปลี่ยน** เพราะจะกระทบทั้งระบบ

```css
--ssk-space-0: 0;
--ssk-space-8: 0.5rem;  /* 8px */
--ssk-space-16: 1rem;   /* 16px */
```

**When to use:** Directly? **Never** (unless creating new semantic tokens)

### Tier 2: Semantic Tokens (ความหมาย)

ค่าที่มีชื่อสื่อความหมาย อ้างอิงถึง Tier 1 ผ่าน `var()`

```css
--ssk-spacing-md: var(--ssk-space-6);  /* 6px spacing */
--ssk-radius-lg: var(--ssk-space-12);  /* 12px radius */
```

**When to use:** **Always** - ใช้ตัวนี้ใน component code

### Benefits

| Scenario | Without 2-Tier | With 2-Tier |
|----------|----------------|-------------|
| Change `md` spacing | Search & replace all `0.375rem` | Change only `--ssk-spacing-md` |
| Scale up spacing | Update every component | Change only primitives |
| Context-specific sizes | Duplicate hardcoded values | Create semantic alias |

---

## Token Reference

### Complete Token Tables

#### 🎯 Spacing Tokens (Tier 2)

ใช้สำหรับ `padding`, `margin`, `gap`

| Semantic Name | Primitive Alias | Rem Value | Px Value | Use Case |
|---------------|-----------------|-----------|----------|----------|
| `--ssk-spacing-none` | `--ssk-space-0` | 0 | 0px | Reset spacing |
| `--ssk-spacing-xxs` | `--ssk-space-1` | 0.0625rem | 1px | Hairline gap |
| `--ssk-spacing-xs` | `--ssk-space-2` | 0.125rem | 2px | Tight spacing |
| `--ssk-spacing-sm` | `--ssk-space-4` | 0.25rem | 4px | Compact spacing |
| `--ssk-spacing-md` | `--ssk-space-6` | 0.375rem | 6px | Medium spacing |
| `--ssk-spacing-lg` | `--ssk-space-8` | 0.5rem | 8px | Large spacing |
| `--ssk-spacing-xl` | `--ssk-space-10` | 0.625rem | 10px | Extra large |
| `--ssk-spacing-2xl` | `--ssk-space-12` | 0.75rem | 12px | Section spacing |
| `--ssk-spacing-3xl` | `--ssk-space-16` | 1rem | 16px | Component gap |
| `--ssk-spacing-4xl` | `--ssk-space-20` | 1.25rem | 20px | Container padding |
| `--ssk-spacing-5xl` | `--ssk-space-24` | 1.5rem | 24px | Large container |
| `--ssk-spacing-6xl` | `--ssk-space-32` | 2rem | 32px | Section gap |
| `--ssk-spacing-7xl` | `--ssk-space-40` | 2.5rem | 40px | Page sections |
| `--ssk-spacing-8xl` | `--ssk-space-44` | 2.75rem | 44px | Major spacing |
| `--ssk-spacing-9xl` | `--ssk-space-48` | 3rem | 48px | Hero spacing |
| `--ssk-spacing-10xl` | `--ssk-space-56` | 3.5rem | 56px | Extra sections |
| `--ssk-spacing-11xl` | `--ssk-space-64` | 4rem | 64px | Massive spacing |
| `--ssk-spacing-12xl` | `--ssk-space-72` | 4.5rem | 72px | Ultra spacing |
| `--ssk-spacing-13xl` | `--ssk-space-80` | 5rem | 80px | Extreme spacing |
| `--ssk-spacing-14xl` | `--ssk-space-96` | 6rem | 96px | Maximum spacing |
| `--ssk-spacing-15xl` | `--ssk-space-128` | 8rem | 128px | Hero sections |

#### ⭕ Border Radius Tokens (Tier 2)

ใช้สำหรับ `border-radius`

| Semantic Name | Primitive Alias | Rem Value | Px Value | Use Case |
|---------------|-----------------|-----------|----------|----------|
| `--ssk-radius-none` | `--ssk-space-0` | 0 | 0px | Square corners |
| `--ssk-radius-xxs` | `--ssk-space-2` | 0.125rem | 2px | Subtle rounding |
| `--ssk-radius-xs` | `--ssk-space-4` | 0.25rem | 4px | Small rounding |
| `--ssk-radius-sm` | `--ssk-space-6` | 0.375rem | 6px | Cards, badges |
| `--ssk-radius-md` | `--ssk-space-8` | 0.5rem | 8px | Buttons, inputs |
| `--ssk-radius-lg` | `--ssk-space-12` | 0.75rem | 12px | Large cards |
| `--ssk-radius-xl` | `--ssk-space-16` | 1rem | 16px | Modal dialogs |
| `--ssk-radius-2xl` | `--ssk-space-20` | 1.25rem | 20px | Hero elements |
| `--ssk-radius-3xl` | `--ssk-space-24` | 1.5rem | 24px | Special cases |
| `--ssk-radius-4xl` | `--ssk-space-32` | 2rem | 32px | Extra rounded |
| `--ssk-radius-full` | `--ssk-space-9999` | 9999px | 9999px | Pill shape, circles |

#### 📏 Border Width Tokens (Tier 2)

ใช้สำหรับ `border-width`

| Semantic Name | Primitive Alias | Rem Value | Px Value | Use Case |
|---------------|-----------------|-----------|----------|----------|
| `--ssk-border-none` | `--ssk-space-0` | 0 | 0px | No border |
| `--ssk-border-xxs` | `--ssk-space-1` | 0.0625rem | 1px | Hairline border |
| `--ssk-border-xs` | `--ssk-space-2` | 0.125rem | 2px | Thin border |
| `--ssk-border-sm` | `--ssk-space-4` | 0.25rem | 4px | Medium border |

#### 🔢 Primitives (Tier 1)

**WARNING:** เหล่านี้คือ Primitive values ใช้เฉพาะเมื่อสร้าง Semantic Tokens ใหม่เท่านั้น

| Name | Rem Value | Px Value |
|------|-----------|----------|
| `--ssk-space-0` | 0 | 0px |
| `--ssk-space-1` | 0.0625rem | 1px |
| `--ssk-space-2` | 0.125rem | 2px |
| `--ssk-space-4` | 0.25rem | 4px |
| `--ssk-space-6` | 0.375rem | 6px |
| `--ssk-space-8` | 0.5rem | 8px |
| `--ssk-space-10` | 0.625rem | 10px |
| `--ssk-space-12` | 0.75rem | 12px |
| `--ssk-space-16` | 1rem | 16px |
| `--ssk-space-20` | 1.25rem | 20px |
| `--ssk-space-24` | 1.5rem | 24px |
| `--ssk-space-32` | 2rem | 32px |
| `--ssk-space-40` | 2.5rem | 40px |
| `--ssk-space-44` | 2.75rem | 44px |
| `--ssk-space-48` | 3rem | 48px |
| `--ssk-space-56` | 3.5rem | 56px |
| `--ssk-space-64` | 4rem | 64px |
| `--ssk-space-72` | 4.5rem | 72px |
| `--ssk-space-80` | 5rem | 80px |
| `--ssk-space-96` | 6rem | 96px |
| `--ssk-space-128` | 8rem | 128px |
| `--ssk-space-9999` | 9999px | 9999px |

---

## Usage Guide

### CSS Usage

#### Basic Usage

```css
/* ✅ CORRECT - Use Semantic Tokens */
.my-component {
  padding: var(--ssk-spacing-md);
  margin: var(--ssk-spacing-lg);
  border-radius: var(--ssk-radius-md);
  border-width: var(--ssk-border-xs);
}

/* ❌ WRONG - Don't use hardcoded values */
.my-component {
  padding: 6px;
  margin: 8px;
  border-radius: 8px;
}
```

#### Shorthand Properties

```css
/* All sides */
.card {
  padding: var(--ssk-spacing-lg);
}

/* Individual sides */
.button {
  padding-top: var(--ssk-spacing-sm);
  padding-right: var(--ssk-spacing-md);
  padding-bottom: var(--ssk-spacing-sm);
  padding-left: var(--ssk-spacing-md);
}

/* Gap for flex/grid */
.container {
  display: flex;
  gap: var(--ssk-spacing-md);
}
```

#### Logical Properties (Recommended)

```css
/* RTL-friendly */
.sidebar {
  padding-inline: var(--ssk-spacing-lg);
  margin-block-start: var(--ssk-spacing-md);
}

/* Same as:
   padding-left & padding-right (LTR)
   padding-right & padding-left (RTL)
*/
```

### Lit/Web Components Usage

```typescript
import { css } from 'lit';

export const myComponentStyles = css`
  :host {
    display: block;
    padding: var(--ssk-spacing-md);
    border-radius: var(--ssk-radius-lg);
  }

  .header {
    margin-bottom: var(--ssk-spacing-lg);
    padding: var(--ssk-spacing-md) var(--ssk-spacing-xl);
  }

  .content {
    gap: var(--ssk-spacing-md);
  }
`;
```

### Dynamic Usage with TypeScript

```typescript
import { cssVar } from '@/types/theme';

// Generate CSS variable name
const spacingVar = cssVar('spacing', 'md'); // Returns: "--ssk-spacing-md"

// Use in template
element.style.padding = `var(${spacingVar})`;
```

---

## TypeScript Integration

### Available Types

```typescript
import type {
  SpacingSize,      // "none" | "xxs" | "xs" | ... | "15xl"
  BorderRadiusSize, // "none" | "xxs" | ... | "full"
  BorderWidthSize   // "none" | "xxs" | "xs" | "sm"
} from '@/types/theme';
```

### Type-Safe Props

```typescript
import type { SpacingSize, BorderRadiusSize } from '@/types/theme';

interface MyComponentProps {
  padding?: SpacingSize;
  borderRadius?: BorderRadiusSize;
}

// ✅ Type-safe
<MyComponent padding="md" borderRadius="lg" />

// ❌ Type error
<MyComponent padding="invalid" borderRadius="wrong" />
```

### Helper Functions

#### `cssVar()` - Generate CSS Variable Names

```typescript
import { cssVar } from '@/types/theme';

// Single key
cssVar('spacing', 'md')     // "--ssk-spacing-md"
cssVar('radius', 'lg')      // "--ssk-radius-lg"
cssVar('border', 'sm')      // "--ssk-border-sm"

// With fallback
const value = `var(${cssVar('spacing', 'md')}, 1rem)`;
```

---

## Best Practices

### ✅ DO's

1. **Always use Semantic Tokens (Tier 2)**
   ```css
   padding: var(--ssk-spacing-md);
   ```

2. **Use Logical Properties for RTL support**
   ```css
   padding-inline: var(--ssk-spacing-lg);
   margin-block: var(--ssk-spacing-md);
   ```

3. **Provide fallbacks when using in dynamic contexts**
   ```css
   padding: var(--ssk-spacing-md, 0.5rem);
   ```

4. **Group related spacing**
   ```css
   .card {
     padding: var(--ssk-spacing-md);       /* horizontal/vertical */
     gap: var(--ssk-spacing-sm);           /* between items */
   }
   ```

5. **Use consistent spacing scale**
   - Use `sm`/`md`/`lg` for most cases
   - Reserve `xxs`/`xs` for tight layouts
   - Use `xl`+ only for section-level spacing

### ❌ DON'Ts

1. **Don't use Primitives directly**
   ```css
   /* ❌ WRONG */
   padding: var(--ssk-space-8);

   /* ✅ CORRECT */
   padding: var(--ssk-spacing-lg);
   ```

2. **Don't mix hardcoded values with tokens**
   ```css
   /* ❌ WRONG */
   padding: var(--ssk-spacing-md) 10px;

   /* ✅ CORRECT */
   padding: var(--ssk-spacing-md) var(--ssk-spacing-lg);
   ```

3. **Don't use arbitrary values**
   ```css
   /* ❌ WRONG */
   margin: 7px;

   /* ✅ CORRECT */
   margin: var(--ssk-spacing-sm); /* or add new token if needed */
   ```

4. **Don't use pixels directly**
   ```css
   /* ❌ WRONG */
   padding: 16px;

   /* ✅ CORRECT */
   padding: var(--ssk-spacing-3xl);
   ```

### Spacing Guidelines by Component

| Component Type | Recommended Spacing |
|----------------|---------------------|
| **Buttons** | `padding: var(--ssk-spacing-sm) var(--ssk-spacing-md)` |
| **Inputs** | `padding: var(--ssk-spacing-sm) var(--ssk-spacing-md)` |
| **Cards** | `padding: var(--ssk-spacing-lg)` |
| **Modals** | `padding: var(--ssk-spacing-xl)` |
| **Sections** | `padding: var(--ssk-spacing-6xl) var(--ssk-spacing-4xl)` |
| **Gap (Flex/Grid)** | `gap: var(--ssk-spacing-md)` |
| **List Items** | `gap: var(--ssk-spacing-sm)` |

---

## Migration Guide

### From Hardcoded Values to Design Tokens

#### Step 1: Identify Hardcoded Values

```bash
# Find hardcoded spacing in your project
grep -r "margin.*px" src/
grep -r "padding.*px" src/
grep -r "border-radius.*px" src/
```

#### Step 2: Map to Tokens

| Old Value | New Token |
|-----------|-----------|
| `0`, `none` | `var(--ssk-spacing-none)` / `var(--ssk-radius-none)` |
| `1px` | `var(--ssk-spacing-xxs)` |
| `2px` | `var(--ssk-spacing-xs)` / `var(--ssk-radius-xxs)` |
| `4px` | `var(--ssk-spacing-sm)` / `var(--ssk-radius-xs)` |
| `6px` | `var(--ssk-spacing-md)` / `var(--ssk-radius-sm)` |
| `8px` | `var(--ssk-spacing-lg)` / `var(--ssk-radius-md)` |
| `10px` | `var(--ssk-spacing-xl)` |
| `12px` | `var(--ssk-spacing-2xl)` / `var(--ssk-radius-lg)` |
| `16px` | `var(--ssk-spacing-3xl)` / `var(--ssk-radius-xl)` |
| `20px` | `var(--ssk-spacing-4xl)` / `var(--ssk-radius-2xl)` |
| `24px` | `var(--ssk-spacing-5xl)` / `var(--ssk-radius-3xl)` |

#### Step 3: Replace in Code

**Before:**
```css
.button {
  padding: 6px 8px;
  border-radius: 8px;
  margin-bottom: 16px;
}
```

**After:**
```css
.button {
  padding: var(--ssk-spacing-sm) var(--ssk-spacing-md);
  border-radius: var(--ssk-radius-md);
  margin-bottom: var(--ssk-spacing-3xl);
}
```

### Common Migration Patterns

#### Pattern 1: Shorthand Padding

```css
/* Before */
padding: 10px 16px;

/* After */
padding: var(--ssk-spacing-xl) var(--ssk-spacing-3xl);
```

#### Pattern 2: Complex Spacing

```css
/* Before */
padding: 12px 20px 12px 20px;

/* After */
padding: var(--ssk-spacing-2xl) var(--ssk-spacing-4xl);
```

#### Pattern 3: Gap Replacement

```css
/* Before */
gap: 8px;

/* After */
gap: var(--ssk-spacing-lg);
```

---

## FAQ

### General Questions

**Q: Why use `rem` instead of `px` for primitives?**

A: `rem` scales with user's browser font size setting, improving accessibility. If base font is 16px:
- `0.5rem` = 8px
- `1rem` = 16px

**Q: Can I use tokens in inline styles?**

A: Yes, but use with caution:
```jsx
<div style={{ padding: 'var(--ssk-spacing-md)' }}>
```

**Q: What if I need a value that's not in the scale?**

A: Two options:
1. **Use closest token** (recommended for consistency)
2. **Create new semantic token** in `design-tokens.css`:
   ```css
   --ssk-spacing-custom: var(--ssk-space-14); /* 14px if needed */
   ```

**Q: How do I use tokens in CSS-in-JS (styled-components, emotion)?**

A: Same as regular CSS:
```javascript
const Button = styled.button`
  padding: var(--ssk-spacing-md);
  border-radius: var(--ssk-radius-lg);
`;
```

**Q: Can I override tokens for a specific component?**

A: Yes, using CSS custom property fallbacks or component-level overrides:
```css
.special-card {
  --ssk-spacing-md: var(--ssk-space-12); /* Override only this scope */
  padding: var(--ssk-spacing-md);
}
```

### Technical Questions

**Q: How do tokens affect bundle size?**

A: Minimal impact. Tokens are plain CSS variables defined once in `:root` and referenced.

**Q: Will this work with SSR (Server-Side Rendering)?**

A: Yes! CSS variables work with SSR as they're standard CSS.

**Q: Can I use these in React Native?**

A: No, React Native doesn't support CSS variables. Use a different approach (like StyleSheet.create).

**Q: What about browser support?**

A: CSS custom properties are supported in all modern browsers:
- Chrome 49+
- Firefox 31+
- Safari 9.1+
- Edge 15+

**Q: How do I debug token values?**

A: Use browser DevTools:
1. Open Elements tab
2. Select element
3. In Computed panel, search for `--ssk-`
4. See resolved values

---

## File Structure

### Where Files Live

```
sellsuki-components/
├── src/
│   ├── assets/
│   │   ├── design-tokens.css    ← Tokens definition (THIS FILE)
│   │   ├── global.css           ← Imports tokens
│   │   └── fonts.css
│   ├── types/
│   │   └── theme.ts             ← TypeScript types
│   └── main.ts                  ← Entry point (imports global.css)
├── .storybook/
│   └── preview-head.html        ← Storybook imports
└── docs/
    └── design-tokens.md         ← This documentation
```

### Import Chain

```
1. design-tokens.css (defines tokens)
       ↓ @import
2. global.css (includes tokens)
       ↓ import
3. main.ts (bundles CSS)
       ↓
4. dist/style.css (output bundle)
```

### Adding New Tokens

To add a new semantic token:

1. **Open `src/assets/design-tokens.css`**
2. **Add to appropriate section:**
   ```css
   /* Example: Add "huge" spacing */
   --ssk-spacing-huge: var(--ssk-space-96);
   ```
3. **Update TypeScript type in `src/types/theme.ts`:**
   ```typescript
   export type SpacingSize =
     | "none" | "xxs" | ... | "15xl"
     | "huge"; // Add new size
   ```
4. **Rebuild:**
   ```bash
   npm run build
   ```

---

## Quick Reference

### Common Usage Cheat Sheet

```css
/* Spacing */
padding: var(--ssk-spacing-md);
margin: var(--ssk-spacing-lg);
gap: var(--ssk-spacing-sm);

/* Border Radius */
border-radius: var(--ssk-radius-md);
border-radius: var(--ssk-radius-full); /* Pill shape */

/* Border Width */
border-width: var(--ssk-border-xs);

/* Combined */
.button {
  padding: var(--ssk-spacing-sm) var(--ssk-spacing-md);
  border-radius: var(--ssk-radius-md);
  border: var(--ssk-border-xs) solid currentColor;
}
```

### Token Naming Convention

```
--ssk-{category}-{size}

Category: spacing | radius | border
Size: none | xxs | xs | sm | md | lg | xl | 2xl | ... | 15xl | full
```

---

## Support & Contributing

### Getting Help

- 📖 View this file: `docs/design-tokens.md`
- 🎨 Check Storybook: See visual examples
- 💬 Ask the team: Design System channel

### Reporting Issues

Found a bug or have a suggestion?
1. Check existing tokens in `design-tokens.css`
2. Verify TypeScript types in `theme.ts`
3. Create issue with examples

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | Mar 2026 | Initial release - Spacing, Border Radius, Border Width tokens |

---

**© 2026 Sellsuki Design System. All rights reserved.**
