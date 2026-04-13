# Sellsuki Components - Service Documentation

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Project Structure](#project-structure)
4. [Design System](#design-system)
5. [Core Systems](#core-systems)
6. [Components](#components)
7. [How Everything Connects](#how-everything-connects)
8. [Development Workflow](#development-workflow)
9. [Usage Guide](#usage-guide)

---

## Overview

**@sellsuki-org/sellsuki-components** เป็น Web Components Library ที่สร้างด้วย **Lit Framework** และ **TypeScript** ออกแบบมาเพื่อให้ใช้งานได้ข้าม Framework (Framework-agnostic) และมี Design System ที่ยืดหยุ่นสูง

### Key Features
- ✅ **Web Components Standard** - ใช้งานได้กับทุก framework (React, Vue, Angular, etc.)
- ✅ **TypeScript** - Full type safety
- ✅ **Design Tokens** - ระบบ theme ที่ยืดหยุ่น
- ✅ **Context Providers** - State management สำหรับ Theme, I18n, Toast
- ✅ **30+ Components** - UI components ครบครัน
- ✅ **Storybook** - Documentation และ testing

---

## Architecture

### Technology Stack

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                        │
│  (React, Vue, Angular, Vanilla JS, etc.)                   │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              Sellsuki Components (Web Components)           │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Lit Framework + TypeScript                         │  │
│  │  - Custom Elements                                  │  │
│  │  - Shadow DOM                                       │  │
│  │  - Reactive Properties                              │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    Core Systems                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ Theme System│  │ I18n System │  │ Toast System│         │
│  │             │  │             │  │             │         │
│  │ CSS Vars    │  │ IndexedDB   │  │ In-Memory   │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

### Build System
- **Vite** - Build tool หลัก
- **Rollup** - Bundling สำหรับ production
- **vite-plugin-dts** - Generate TypeScript declaration files
- **Storybook** - Component documentation

---

## Project Structure

```
sellsuki-components/
├── src/
│   ├── main.ts                    # Entry point, export ทุกอย่าง
│   ├── assets/                    # Static assets
│   │   ├── design-tokens.css      # CSS Variables (Tier 1)
│   │   ├── global.css             # Global styles
│   │   └── fonts.css              # Font definitions
│   │
│   ├── types/                     # TypeScript definitions
│   │   └── theme.ts               # Theme types & utilities
│   │
│   ├── contexts/                  # State Management
│   │   ├── theme/                 # Theme provider
│   │   │   ├── index.ts           # ThemeProvider component
│   │   │   └── default.ts         # Default theme config
│   │   ├── i18n/                  # Internationalization
│   │   │   ├── index.ts           # I18nProvider component
│   │   │   └── idb.ts             # IndexedDB storage
│   │   └── toast/                 # Toast notifications
│   │       ├── index.ts           # ToastProvider component
│   │       └── in-memory.ts       # In-memory storage
│   │
│   ├── elements/                  # Atomic UI Components
│   │   ├── button/                # Button component
│   │   ├── input/                 # Input component
│   │   ├── checkbox/              # Checkbox component
│   │   ├── ...                    # (30+ atomic components)
│   │   └── widgets/               # Dashboard widgets
│   │
│   └── components/                # Composite Components
│       ├── modal/                 # Modal component
│       ├── dropdown/              # Dropdown component
│       ├── calendar/              # Calendar component
│       └── ...                    # (Complex components)
│
├── .storybook/                    # Storybook configuration
├── docs/                          # Design system documentation
├── scripts/                       # Generator scripts
├── vite.config.js                 # Vite configuration
├── package.json                   # Dependencies
└── tsconfig.json                  # TypeScript config
```

---

## Design System

### 2-Tier Architecture

```
┌────────────────────────────────────────────────────────────┐
│                    TIER 2: Semantic Tokens                 │
│                   (ใช้ใน Components)                       │
│                                                             │
│  --ssk-spacing-md  →  var(--ssk-space-6)                   │
│  --ssk-radius-lg   →  var(--ssk-space-12)                  │
│  --ssk-border-sm   →  var(--ssk-space-4)                   │
└────────────────────┬───────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────┐
│                    TIER 1: Primitives                       │
│                    (ค่าพื้นฐาน)                          │
│                                                             │
│  --ssk-space-0    →  0                                     │
│  --ssk-space-1    →  0.0625rem (1px)                       │
│  --ssk-space-2    →  0.125rem (2px)                        │
│  --ssk-space-4    →  0.25rem (4px)                         │
│  ...                                                         │
└────────────────────────────────────────────────────────────┘
```

### Design Tokens Categories

#### 1. **Spacing System** (16 levels)
```css
--ssk-spacing-none   → 0
--ssk-spacing-xxs    → 1px
--ssk-spacing-xs     → 2px
--ssk-spacing-sm     → 4px
--ssk-spacing-md     → 6px
--ssk-spacing-lg     → 8px
--ssk-spacing-xl     → 10px
--ssk-spacing-2xl    → 12px
--ssk-spacing-3xl    → 16px
...
--ssk-spacing-15xl   → 128px
```

#### 2. **Border Radius** (11 levels)
```css
--ssk-radius-none    → 0
--ssk-radius-xxs     → 2px
--ssk-radius-xs      → 4px
--ssk-radius-sm      → 6px
--ssk-radius-md      → 8px
...
--ssk-radius-full    → 9999px
```

#### 3. **Color Palette** (60+ colors)

**Semantic Colors:**
```css
--ssk-colors-primary-500      → หลัก
--ssk-colors-secondary-500    → รอง
--ssk-colors-success-500      → สำเร็จ
--ssk-colors-warning-500      → เตือน
--ssk-colors-error-500        → ผิดพลาด
--ssk-colors-info-500         → ข้อมูล
```

**Color Scales (50-1000):**
```css
--ssk-colors-blue-50      → สว่างสุด
--ssk-colors-blue-500     → ปกติ
--ssk-colors-blue-900     → มืดสุด
```

#### 4. **Typography**
```css
--ssk-font-size-xs    → 18px
--ssk-font-size-sm    → 20px
--ssk-font-size-md    → 24px
...
--ssk-font-weight-normal  → 400
--ssk-font-weight-bold    → 700
```

#### 5. **Animations**
```css
--ssk-animation-spin     → หมุน
--ssk-animation-pulse    → กระพริบ
--ssk-animation-bounce   → กระเด้ง
```

---

## Core Systems

### 1. Theme System

**ไฟล์หลัก:** `src/contexts/theme/`

#### Flow:
```
default.ts (Theme Config)
    ↓
ThemeProvider (Component)
    ↓
parseThemeToCssVariables()
    ↓
CSS Variables in :host
    ↓
Components consume via @consume()
```

#### วิธีการทำงาน:

1. **Theme Definition** (`default.ts`)
```typescript
export const defaultTheme: Theme = {
  colors: { ... },
  fontSize: { ... },
  components: {
    button: { padding: { ... } },
    input: { rounded: { ... } },
    // Component-specific overrides
  }
};
```

2. **ThemeProvider** (`index.ts`)
```typescript
@customElement("ssk-theme-provider")
export class ThemeProvider extends LitElement {
  @provide({ context: themeContext })
  theme: Theme = defaultTheme;

  render() {
    return html`
      ${parseThemeToCssVariables(this.theme, ":host")}
      <slot></slot>
    `;
  }
}
```

3. **Component Consumption** (`elements/button/index.ts`)
```typescript
@consume({ context: themeContext })
@property({ attribute: false })
public theme?: Theme;

render() {
  // Access component-specific theme
  const buttonTheme = this.theme?.components?.button;
  // Apply CSS variables
  return html`
    ${parseThemeToCssVariables(buttonTheme, "button")}
    <button>...</button>
  `;
}
```

4. **CSS Variable Fallback System**
```typescript
// ตัวอย่างการ fallback
const color = parseVariables(
  cssVar("colors", "primary", 500),  // 1. Component-specific
  cssVar("colors", "primary"),        // 2. Color palette
  "#44aef8"                            // 3. Default value
);
// → var(--ssk-colors-primary-500, var(--ssk-colors-primary, #44aef8))
```

---

### 2. I18n System (Internationalization)

**ไฟล์หลัก:** `src/contexts/i18n/`

#### Flow:
```
I18nProvider (Component)
    ↓
IdbI18nStore (IndexedDB Storage)
    ↓
Translation Keys
    ↓
<ssk-i18n-template> / <ssk-i18n-translate>
```

#### วิธีการทำงาน:

1. **I18nProvider Setup**
```typescript
@customElement("ssk-i18n-provider")
export class I18nProvider extends LitElement {
  @provide({ context: i18nContext })
  store: I18nStore = new IdbI18nStore();

  @property()
  lang: string = "en";  // ภาษาปัจจุบัน
}
```

2. **IndexedDB Storage** (`idb.ts`)
- เก็บ translations ใน IndexedDB (offline support)
- Support multiple languages
- Fallback language system

3. **Usage in Components**
```html
<!-- Template rendering -->
<ssk-i18n-template
  key="welcome_message"
  metadata='{ "name": "John" }'
></ssk-i18n-template>

<!-- Simple translation -->
<ssk-i18n-translate
  key="submit_button"
></ssk-i18n-translate>
```

---

### 3. Toast System

**ไฟล์หลัก:** `src/contexts/toast/`

#### Flow:
```
ToastProvider (Container)
    ↓
InMemoryStore (State Management)
    ↓
ssk-toast (Component)
    ↓
Auto-dismiss (setTimeout)
```

#### วิธีการทำงาน:

1. **ToastProvider**
```typescript
@customElement("ssk-toast-provider")
export class ToastProvider extends LitElement {
  @provide({ context: toastContext })
  toast: ToastStore = {
    toasts: [],
    addToast: (data) => { ... },
    removeToast: (id) => { ... },
    clearToasts: () => { ... }
  };

  // Poll every 100ms for updates
  private poller = setInterval(() => {
    this.requestUpdate();
  }, 100);
}
```

2. **Toast Types**
```typescript
type ToastType = "success" | "error" | "info" | "warning";

interface ToastData {
  id: string;
  title: string;
  message: string;
  type: ToastType;
  timeout: number;
}
```

3. **Usage**
```typescript
// In any component
@consume({ context: toastContext })
toast!: ToastStore;

// Show toast
this.toast.addToast({
  title: "Success!",
  message: "Data saved successfully",
  type: "success",
  timeout: 3000
});
```

---

## Components

### Component Categories

#### 1. **Atomic Elements** (`src/elements/`)
Building blocks พื้นฐาน:

**Form Elements:**
- `<ssk-button>` - ปุ่มกด (4 variants: solid, outline, ghost, solid-light)
- `<ssk-input>` - ช่องกรอกข้อมูล
- `<ssk-textarea>` - ช่องกรอกข้อความหลายบรรทัด
- `<ssk-checkbox>` - ช่องทำเครื่องหมาย
- `<ssk-radio>` - ปุ่มเลือกแบบ radio
- `<ssk-toggle>` - สลับ on/off
- `<ssk-pin-code>` - ช่องกรอก PIN code

**Content Elements:**
- `<ssk-text>` - ข้อความ
- `<ssk-heading>` - หัวข้อ
- `<ssk-divider>` - เส้นแบ่ง
- `<ssk-container>` - คอนเทนเนอร์
- `<ssk-image>` - รูปภาพ
- `<ssk-avatar>` - รูปโปรไฟล์
- `<ssk-icon>` - ไอคอน
- `<ssk-code-block>` - โค้ดบล็อก

**Feedback Elements:**
- `<ssk-alert>` - แจ้งเตือน
- `<ssk-badge>` - ป้าย
- `<ssk-tag>` - แท็ก
- `<ssk-spinner>` - หมุนๆ
- `<ssk-progress-bar>` - แถบความคืบหน้า

**Navigation:**
- `<ssk-tabs>` - แท็บ
- `<ssk-menu>` - เมนู
- `<ssk-accordion>` - accordion

**Data Display:**
- `<ssk-table>` - ตาราง
- `<ssk-card>` - การ์ด

**Specialized:**
- `<ssk-date-display>` - แสดงวันที่
- `<ssk-top-navbar>` - แถบนำทางด้านบน

#### 2. **Composite Components** (`src/components/`)
Components ที่ซับซ้อน:

**Layout:**
- `<ssk-modal>` - Modal popup
- `<ssk-drawer>` - ลิ้นชัก
- `<ssk-sidebar>` - แถบข้าง
- `<ssk-card-expandable>` - การ์ดขยายได้

**Data:**
- `<ssk-dynamic-table>` - ตารางแบบ dynamic
- `<ssk-pagination>` - แบ่งหน้า
- `<ssk-timeline>` - Timeline

**Input:**
- `<ssk-calendar>` - ปฏิทิน
- `<ssk-date-picker>` - เลือกวันที่
- `<ssk-time>` - เลือกเวลา
- `<ssk-dropdown>` - Dropdown
- `<ssk-addon-phone-country>` - กรอกเบอร์โทร + country code

**Feedback:**
- `<ssk-skeleton>` - Skeleton loading
- `<ssk-tooltip>` - คำแนะนำ
- `<ssk-stepper>` - ขั้นตอน

**Special:**
- `<ssk-image-cropper>` - ครอปรูป
- `<ssk-widget-grid>` - Grid สำหรับ widgets
- `<ssk-download-file>` - ดาวน์โหลดไฟล์

#### 3. **Widget Components** (`src/elements/widgets/`)
สำหรับ Dashboard:

- `<ssk-widget-table>` - ตารางใน widget
- `<ssk-widget-matric>` - Metric/KPI
- `<ssk-widget-user-detail>` - ข้อมูล user
- `<ssk-widget-title>` - หัวข้อ widget
- `<ssk-widget-example>` - ตัวอย่าง widget

---

## How Everything Connects

### Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Developer Code                           │
│                                                             │
│  import { Button, ThemeProvider, defaultTheme }            │
│    from '@sellsuki-org/sellsuki-components'                │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                  Application Setup                          │
│                                                             │
│  <ssk-theme-provider .theme=${customTheme}>                │
│    <ssk-i18n-provider lang="th">                           │
│      <ssk-toast-provider>                                  │
│        <!-- Your App Here -->                               │
│      </ssk-toast-provider>                                 │
│    </ssk-i18n-provider>                                    │
│  </ssk-theme-provider>                                     │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                 Component Rendering                         │
│                                                             │
│  <ssk-button                                               │
│    themeColor="primary"                                    │
│    size="lg"                                               │
│    variant="solid"                                         │
│  >Click Me</ssk-button>                                    │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              Component Internal Processing                  │
│                                                             │
│  1. @consume(themeContext) → Get theme                     │
│  2. Parse component-specific theme                         │
│  3. Calculate CSS variables with fallbacks                 │
│  4. Apply to Shadow DOM                                    │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    CSS Variables                            │
│                                                             │
│  :host {                                                    │
│    --ssk-colors-primary-500: #44aef8;                      │
│    --ssk-font-size-lg: 26px;                               │
│    --ssk-padding-lg: 8px 16px;                             │
│    --ssk-rounded-md: 6px;                                  │
│  }                                                          │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                   Final Render                              │
│                                                             │
│  <button class="ssk-button">                               │
│    → Styled with CSS variables                             │
│    → Scoped with Shadow DOM                                │
│    → Reactive with Lit                                      │
│  </button>                                                  │
└─────────────────────────────────────────────────────────────┘
```

### Component Lifecycle

```
1. DEFINITION (@customElement)
   ↓
2. REGISTRATION (customElements.define)
   ↓
3. INSTANTIATION (new element created)
   ↓
4. PROPERTY SETTING (@property values set)
   ↓
5. CONTEXT INJECTION (@consume theme)
   ↓
6. RENDER (render() method called)
   ↓
7. STYLES (static styles applied)
   ↓
8. SHADOW DOM ATTACHMENT
   ↓
9. REACTIVE UPDATES (property changes → re-render)
```

---

## Development Workflow

### Build Process

```
1. SOURCE CODE
   src/main.ts → All components & contexts
   ↓
2. VITE BUILD
   - Bundles with Rollup
   - Externalizes Lit
   - Generates declarations
   ↓
3. OUTPUT
   dist/sellsuki-components.js        (ES Module)
   dist/sellsuki-components.umd.cjs    (UMD)
   dist/main.d.ts                      (TypeScript types)
   dist/style.css                      (Styles)
```

### Development Scripts

```bash
# Start dev server
npm run dev

# Build for production
npm run build

# Run Storybook
npm run storybook

# Run tests
npm test

# Generate icons from SVG
npm run generate:icon

# Generate font files
npm run generate:font

# Generate country codes
npm run generate:country
```

---

## Usage Guide

### Installation

```bash
npm install @sellsuki-org/sellsuki-components
```

### Basic Setup

```typescript
// 1. Import components
import {
  ThemeProvider,
  I18nProvider,
  ToastProvider,
  Button,
  Input,
  Modal
} from '@sellsuki-org/sellsuki-components';

// 2. Import styles
import '@sellsuki-org/sellsuki-components/style.css';
```

### HTML Usage

```html
<!-- Wrap with providers -->
<ssk-theme-provider>
  <ssk-i18n-provider lang="th">
    <ssk-toast-provider>

      <!-- Use components -->
      <ssk-button
        themeColor="primary"
        size="lg"
        variant="solid">
        คลิกฉัน
      </ssk-button>

      <ssk-input
        placeholder="กรอกชื่อ..."
        size="md">
      </ssk-input>

    </ssk-toast-provider>
  </ssk-i18n-provider>
</ssk-theme-provider>
```

### Custom Theme

```typescript
import { applyTheme } from '@sellsuki-org/sellsuki-components';

// Create custom theme
const myTheme = applyTheme({
  colors: {
    primary: {
      500: '#ff6b6b',
      600: '#ee5a52',
    }
  },
  components: {
    button: {
      padding: {
        md: '12px 24px'  // Override button padding
      }
    }
  }
});

// Apply to provider
document.querySelector('ssk-theme-provider').theme = myTheme;
```

### TypeScript Support

```typescript
// Full type safety
import type { Theme, ColorRole, Size } from '@sellsuki-org/sellsuki-components';

const buttonColor: ColorRole = 'primary';
const buttonSize: Size = 'lg';
```

---

## Key Patterns

### 1. Component Pattern

```typescript
@customElement("ssk-my-component")
export class MyComponent extends LitElement {
  // 1. Consume context
  @consume({ context: themeContext })
  theme?: Theme;

  // 2. Define properties
  @property({ type: String })
  size: Size = "md";

  // 3. Render with theme
  render() {
    return html`
      ${parseThemeToCssVariables(this.theme?.components?.myComponent)}
      <div class="my-component">
        <slot></slot>
      </div>
    `;
  }

  // 4. Scoped styles
  static styles = css`
    :host {
      display: block;
    }
    .my-component {
      font-size: var(--ssk-font-size);
      padding: var(--ssk-padding);
    }
  `;
}
```

### 2. CSS Variable Fallback

```typescript
// Always provide fallbacks
const padding = parseVariables(
  cssVar("padding", this.padding),      // 1. User prop
  cssVar("padding", this.size),         // 2. Size prop
  cssVar("spacing", "md")               // 3. Default
);
```

### 3. Slot Pattern

```html
<!-- Named slots for flexibility -->
<ssk-button>
  <div slot="prefix">👈</div>
  Button Text
  <div slot="postfix">👉</div>
</ssk-button>
```

---

## Performance Optimizations

1. **Lit's Reactive Rendering** - Only re-renders when properties change
2. **Shadow DOM Scoping** - Styles don't leak
3. **CSS Variables** - Dynamic updates without JS
4. **Lazy Loading** - Components load on demand
5. **Tree Shaking** - Unused code removed in build

---

## Summary

### Architecture Flow
```
Design Tokens (CSS)
    ↓
TypeScript Types
    ↓
Theme Provider (Context)
    ↓
Components (@consume)
    ↓
CSS Variables (Fallback Chain)
    ↓
Shadow DOM (Scoped Styles)
    ↓
Final Rendered Component
```

### Key Takeaways

1. **Framework Agnostic** - ใช้กับ framework ไหนก็ได้
2. **Design Tokens** - Centralized design system
3. **Context Providers** - Shared state across components
4. **Shadow DOM** - Style isolation
5. **TypeScript** - Full type safety
6. **Lit Framework** - Lightweight & fast

---

## Quick Reference

### Import Paths
```typescript
// Everything
import * as SSK from '@sellsuki-org/sellsuki-components';

// Specific
import { Button, Input } from '@sellsuki-org/sellsuki-components';
import { ThemeProvider } from '@sellsuki-org/sellsuki-components/contexts/theme';
import { defaultTheme } from '@sellsuki-org/sellsuki-components/contexts/theme/default';
```

### CSS Variables Naming
```css
--ssk-{category}-{name}-{scale}
```

Examples:
- `--ssk-colors-primary-500`
- `--ssk-spacing-md`
- `--ssk-radius-lg`

### Component Props Pattern
```typescript
// Base props
testId?: string;

// Theme props
themeColor?: ColorRole | ColorName;
size?: Size;

// Component-specific
variant?: ButtonVariants;
disabled?: boolean;
```

---

**Last Updated:** March 2026
**Version:** 0.1.2
**License:** See package.json
