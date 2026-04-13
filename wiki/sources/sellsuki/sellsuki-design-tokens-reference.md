---
title: "Sellsuki Design Tokens — Spacing System Reference"
type: source
source_file: raw/notes/central-component/design-tokens.md
url: ""
published: 2026-03-01
tags: [design-tokens, spacing, border-radius, css-variables, sellsuki, reference]
related: [wiki/sources/sellsuki-design-tokens-service.md, wiki/concepts/design-tokens.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/central-component/design-tokens.md|Original file]]

## สรุป

Official documentation ของ Sellsuki Design Tokens Spacing System — อธิบาย 2-Tier Architecture (Primitives + Semantic), token reference ครบ (spacing 21 levels, border-radius 11 levels, border-width 4 levels), usage guide, best practices และ migration guide จาก hardcoded values

## ประเด็นสำคัญ

- **2-Tier Architecture**: Primitives (Tier 1, `--ssk-space-*`) → Semantic Tokens (Tier 2, `--ssk-spacing-*` / `--ssk-radius-*` / `--ssk-border-*`) — ใช้แค่ Tier 2 ใน code
- **Semantic tokens ไม่ควรใช้ Primitives โดยตรง** — เสมอใช้ semantic alias ใน component
- **ทำไม rem?** Scales กับ browser font size → accessibility
- **3 categories**: Spacing (21 levels), Border Radius (11 levels), Border Width (4 levels)
- **CSS-in-JS supported**: ใช้ `var(--ssk-spacing-md)` ได้ใน styled-components, emotion ปกติ
- **SSR-compatible**: CSS variables เป็น standard CSS ทำงานได้กับ SSR

## ข้อมูล / หลักฐาน ที่น่าสนใจ

**Token ที่ใช้บ่อยสุด:**
| Token | ค่า | ใช้เมื่อ |
|-------|-----|---------|
| `--ssk-spacing-sm` | 4px | Compact spacing, button padding |
| `--ssk-spacing-md` | 6px | Medium spacing |
| `--ssk-spacing-lg` | 8px | Large spacing |
| `--ssk-radius-md` | 8px | Buttons, inputs |
| `--ssk-radius-full` | 9999px | Pill shape, circles |
| `--ssk-border-xs` | 2px | Thin border |

**Mapping จาก px เก่า:**
| Pixel เก่า | Token ใหม่ |
|-----------|-----------|
| 4px | `--ssk-spacing-sm` / `--ssk-radius-xs` |
| 6px | `--ssk-spacing-md` / `--ssk-radius-sm` |
| 8px | `--ssk-spacing-lg` / `--ssk-radius-md` |
| 16px | `--ssk-spacing-3xl` / `--ssk-radius-xl` |

**Spacing Guidelines:**
- Buttons: `padding: var(--ssk-spacing-sm) var(--ssk-spacing-md)`
- Cards: `padding: var(--ssk-spacing-lg)`
- Modals: `padding: var(--ssk-spacing-xl)`

**Browser Support:** Chrome 49+, Firefox 31+, Safari 9.1+, Edge 15+

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่มีข้อขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/design-tokens|Design Tokens]]
