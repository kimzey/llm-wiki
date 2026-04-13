---
title: "Backup Strategy"
type: concept
tags: [backup, 3-2-1-rule, disaster-recovery, rclone]
sources: [wiki/sources/homeserver-admin-knowledge.md]
related: []
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

3-2-1 Backup Rule คือมาตรฐานสากลของการ backup — 3 copies, 2 media types, 1 off-site — ร่วมกับ automated scripts และ cloud sync เพื่อป้องกัน data loss จาก hardware failure, theft, fire, หรือ ransomware

## อธิบาย

Backup ที่ดีต้องมี **redundancy** (หลาย copies) และ **separation** (แยกกัน) — เพื่อไม่ให้ single point of failure ทำลายข้อมูลทั้งหมด 3-2-1 rule คือ guideline ที่ง่ายที่สุดจำ

## ประเด็นสำคัญ

### 3-2-1 Rule Explained

```
3 copies ข้อมูล
  ├── 1. Primary data ในเครื่อง NUC (/opt/docker)
  ├── 2. Backup ใน HDD External (/srv/nas/backup)
  └── 3. Backup บน Cloud (Google Drive / Backblaze B2)

2 media ต่างกัน
  ├── SSD ใน NUC
  └── HDD External (ถ้า SSD เสีย HDD ยังอยู่)

1 off-site copy
  └── Cloud storage (ป้องกัน fire, theft, flood)
```

**ทำไมต้อง 3 copies?**
- 1 copy — ไม่มี backup เลย
- 2 copies — redundancy น้อยเกินไป
- 3 copies — **minimal safe** level

**ทำไมต้อง 2 media types?**
- ถ้า SSD เสีย — HDD external ยังอยู่
- ทำไมไม่ใช้ SSD 2 ตัว? — same failure mode (อายุใกล้เคียงกัน)

**ทำไมต้อง 1 off-site?**
- **Fire, theft, flood** ทำลายทุกอย่างในบ้าน
- Ransomware อาจ encrypt network drives ทั้งหมด
- Cloud คือ insurance สุดท้าย

### Backup Script

```bash
#!/usr/bin/bash
# /usr/local/bin/backup.sh
set -e                           # exit ถ้ามี error

BACKUP_ROOT="/srv/nas/backup"
DATE=$(date +%Y-%m-%d_%H%M)
LOG="/var/log/backup.log"

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG"
}

log "=== เริ่ม Backup ==="

# 1. หยุด database containers ก่อน
# (ป้องกัน data corruption ระหว่าง backup)
log "หยุด database containers..."
docker stop postgres 2>/dev/null || true

# 2. Backup Docker volumes
log "Backup /opt/docker..."
mkdir -p "$BACKUP_ROOT/docker"
tar -czf "$BACKUP_ROOT/docker/docker-$DATE.tar.gz" \
  --exclude='/opt/docker/netdata' \
  /opt/docker/

# 3. เริ่ม containers กลับ
log "เริ่ม database containers..."
docker start postgres 2>/dev/null || true

# 4. Backup database โดยตรง
# (ดีกว่า backup ไฟล์ เพราะมี transaction integrity)
log "Backup PostgreSQL..."
mkdir -p "$BACKUP_ROOT/db"
docker exec postgres pg_dumpall -U postgres \
  > "$BACKUP_ROOT/db/postgres-$DATE.sql"
gzip "$BACKUP_ROOT/db/postgres-$DATE.sql"

# 5. ลบ backup เก่ากว่า 30 วัน
log "ลบ backup เก่า..."
find "$BACKUP_ROOT" -name "*.tar.gz" -mtime +30 -delete
find "$BACKUP_ROOT" -name "*.sql.gz" -mtime +30 -delete

# 6. แสดงผล
BACKUP_SIZE=$(du -sh "$BACKUP_ROOT" | cut -f1)
log "=== Backup สำเร็จ! ขนาดรวม: $BACKUP_SIZE ==="
```

### Crontab — Schedule

```bash
crontab -e

# เพิ่ม:
0 2 * * *     /usr/local/bin/backup.sh
# รันทุกวัน เวลา 02:00 น.
```

### Cloud Backup ด้วย rclone

**rclone** คือ rsync for cloud — sync ไป Google Drive, Backblaze B2, S3 ฯลฯ

```bash
# ติดตั้ง
curl https://rclone.org/install.sh | sudo bash

# ตั้งค่า (interactive)
rclone config
# → New remote
# → Name: gdrive
# → Type: Google Drive
# → Follow OAuth flow

# ทดสอบ sync
rclone sync /srv/nas/backup gdrive:homeserver-backup --dry-run

# Sync จริง
rclone sync /srv/nas/backup gdrive:homeserver-backup \
  --log-file=/var/log/rclone.log \
  --log-level INFO

# เพิ่มใน backup script
# (ต่อท้าย script หลังจาก local backup เสร็จ)
rclone sync /srv/nas/backup gdrive:homeserver-backup \
  --log-file=/var/log/rclone.log
```

### What to Backup?

**ต้อง Backup:**
- ✅ Docker volumes (`/opt/docker`)
- ✅ Database dumps (`pg_dump`, `mysqldump`)
- ✅ Configuration files (`docker-compose.yml`, `.env` examples)
- ✅ Application data ที่ generate เอง

**ไม่ต้อง Backup:**
- ❌ Docker images (pull ได้ใหม่)
- ❌ Temporary files, cache, logs
- ❌ OS files ( reinstall ได้)

### Backup Strategies ต่างๆ

| Strategy | Pros | Cons | Use Case |
|----------|------|------|----------|
| **Full** | เร็ว restore | ช้า, กินพื้นที่ | Small datasets |
| **Incremental** | เร็ว, ประหยัด | ช้า restore | Large datasets |
| **Differential** | balance | กินพื้นที่ | Medium |
| **Snapshot** (ZFS/Btrfs) | เร็วมาก | ต้อง filesystem | Advanced |

### Best Practices

**1. Test Regularly**
```bash
# ทดสอบ restore เดือนละครั้ง
docker stop postgres
tar -xzf backup.tar.gz
docker exec postgres psql < backup.sql
# verify data integrity
```

**2. Encryption**
```bash
# Encrypt backup ก่อน upload cloud
gpg --encrypt --recipient you@email.com backup.tar.gz
```

**3. Monitoring**
```bash
# Script แจ้งเตือนถ้า backup fail หรือไม่รันนานเกินไป
LAST_BACKUP=$(stat -c %Y /srv/nas/backup/docker/latest.tar.gz)
NOW=$(date +%s)
AGE=$((NOW - LAST_BACKUP))
if [ $AGE -gt 172800 ]; then  # 48 ชั่วโมง
  echo "Backup ล้าสุด!" | mail -s "Alert" admin@email.com
fi
```

**4. Retention Policy**
```bash
# รักษา backup ย้อนหลัง
- Daily backups: 7 วัน
- Weekly backups: 4 สัปดาห์
- Monthly backups: 12 เดือน
```

### Backup Locations Comparison

| Location | Cost | Speed | Security | Off-site? |
|----------|------|-------|----------|-----------|
| **Local SSD** | Free | ⚡️ Fast | ✅ Full control | ❌ No |
| **External HDD** | Cheap | 🚀 Fast | ✅ Full control | ⚠️ Manual |
| **NAS (Synology)** | Medium | ⚡️ Fast | ✅ Full control | ⚠️ Manual |
| **Google Drive** | Free/cheap | 🐢 Medium | ⚠️ Cloud | ✅ Yes |
| **Backblaze B2** | Cheap | 🐌 Slow | ✅ Encrypted | ✅ Yes |
| **AWS S3 Glacier** | Medium | 🐌 Very slow | ✅ Encrypted | ✅ Yes |

### Recovery Scenarios

**Scenario 1: Single container พัง**
```bash
docker logs container              # ดูว่าทำไมพัง
docker restart container           # restart
docker compose up -d --force-recreate  # recreate
# restore volume ถ้าจำเป็น
```

**Scenario 2: Disk เสีย**
```bash
# 1. ใส่ disk ใหม่
# 2. Install Ubuntu + Docker
# 3. Copy backup จาก external HDD
tar -xzf backup.tar.gz -C /opt/docker/
# 4. Restore database
docker exec postgres psql < backup.sql
# 5. Start containers
docker compose up -d
```

**Scenario 3: Ransomware / Fire**
```bash
# 1. ซื้อ hardware ใหม่
# 2. Download จาก cloud
rclone sync gdrive:homeserver-backup /srv/nas/restore
# 3. Restore จาก backup (ตาม scenario 2)
```

## Common Mistakes

❌ **Backup แต่ไม่เคย test restore** — backup พังก็ไม่รู้

❌ **Backup password file แต่ลืม key** — encrypt แล้วหา key ไม่เจอ

❌ **ใส่ทุกอย่างใน backup** — สิ้นเปลือง, restore ช้า

❌ **Backup ไว้ในเครื่องเดียวกัน** — disk เสีย = หมดเกม

❌ **ลืม backup database** แต่ backup volume เฉยๆ — data inconsistent

## Tools Comparison

| Tool | Type | Complexity | Best For |
|------|------|------------|----------|
| **tar** | Archive | ⭐ Simple | Local backups |
| **rsync** | Sync | ⭐ Simple | Local/remote sync |
| **rclone** | Cloud sync | ⭐⭐ Medium | Cloud backups |
| **restic** | Deduplicated backup | ⭐⭐⭐ Advanced | Large datasets |
| **Borg** | Deduplicated + encrypted | ⭐⭐⭐ Advanced | Secure backups |
| **Duplicati** | GUI backup | ⭐ Simple | Non-technical users |

## Checklist

### Setup
- [ ] Backup script สร้างแล้ว
- [ ] External HDD เชื่อมต่อแล้ว
- [ ] rclone config แล้ว (cloud)
- [ ] Crontab ตั้งแล้ว
- [ ] Test restore สำเร็จแล้ว

### Ongoing
- [ ] Check backup logs ทุกสัปดาห์
- [ ] Test restore ทุกเดือน
- [ ] Review retention policy ทุกไตรมาส
- [ ] Update backup script เมื่อเพิ่ม service

## แหล่งเรียนรู้เพิ่มเติม

- **rclone.org** — cloud sync tool
- **restic.net** — modern backup tool
- **borgbackup.org** — deduplication backup
- **The 3-2-1 Strategy** — backblaze.com/blog/backup-strategy-3-2-1
