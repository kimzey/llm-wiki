---
name: wiki-status
description: Show a quick dashboard of the wiki state — page counts, recent activity, empty sections, and suggested next actions.
origin: user
---

# Wiki Status Dashboard

Vault root: `/Users/kimzey/Desktop/local-valut/`

## Steps

### 1. นับหน้าทุกประเภท
```
Glob wiki/concepts/**/*.md → count
Glob wiki/books/**/*.md → count
Glob wiki/sources/**/*.md → count
Glob wiki/synthesis/**/*.md → count
Glob raw/**/*.md → count (ยกเว้น .gitkeep)
```

### 2. อ่าน log.md
แสดง 5 entries ล่าสุด

### 3. อ่าน index.md
ตรวจว่า index ครบถ้วน

### 4. Output Dashboard

```markdown
# Wiki Status — [YYYY-MM-DD HH:MM]

## หน้าทั้งหมด
| ประเภท | จำนวน |
|--------|-------|
| Concepts | N |
| Books | N |
| Sources | N |
| Synthesis | N |
| **รวม** | **N** |

## Raw Sources
| ประเภท | จำนวน |
|--------|-------|
| Clips | N |
| Books | N |
| Notes | N |

## กิจกรรมล่าสุด (5 entries)
[จาก log.md]

## สิ่งที่ควรทำต่อ
- [ ] ถ้า raw มีไฟล์ที่ยังไม่ ingest → แสดงชื่อ
- [ ] ถ้า wiki ว่างเปล่า → แนะนำให้เริ่ม ingest
- [ ] ถ้า lint ยังไม่ได้รัน > 7 วัน → แนะนำ /lint
```
