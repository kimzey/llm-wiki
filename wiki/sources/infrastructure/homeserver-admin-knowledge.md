---
title: "Home Server Admin — ความรู้ที่ต้องรู้"
type: source
source_file: raw/notes/homeserver-knowledge.md
tags: [linux, docker, security, backup, monitoring, systemd]
related: []
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/homeserver-knowledge.md|Original file]]

## สรุป

คู่มือครอบคลุมทุกเรื่องที่จะเจอจริงๆ ในการดูแล Home Server (Intel NUC + Ubuntu 24.04 + Docker) — เริ่มตั้งแต่ Linux command line พื้นฐาน, Docker, Security, Backup, Monitoring, ไปจนถึง Troubleshooting และ Production Checklist

## ประเด็นสำคัญ

### ส่วนที่ 1: Linux Command Line พื้นฐาน
- **การจัดการไฟล์**: `pwd`, `ls`, `cd`, `mkdir`, `cp`, `mv`, `rm`, `cat`, `less`, `head`, `tail`
- **การแก้ไขไฟล์**: nano editor (Ctrl+O save, Ctrl+X exit)
- **Permission**: `chmod` (rwx permission), `chown` (เปลี่ยนเจ้าของ)
- **Process Management**: `ps`, `htop`, `kill`, `killall`, รัน background (`&`, `nohup`)
- **Pipe & Redirect**: `|` (ส่ง output ต่อ), `>` (เขียนทับ), `>>` (เพิ่มต่อท้าย), `2>&1 | tee`
- **Network Commands**: `ip a`, `ping`, `ss -tlnp`, `ip route`, `nslookup`, `dig`
- **Disk & Memory**: `df -h`, `du -sh`, `free -h`, `iostat`

### ส่วนที่ 2: Docker Commands
- **Container Management**: `docker ps`, `start/stop/restart`, `logs -f`, `exec -it`, `inspect`, `stats`
- **Docker Compose**: `up -d`, `down`, `pull`, `logs -f`, `restart`
- **Image Management**: `docker images`, `pull`, `rmi`, `system prune`
- **Volume & Network**: `docker volume ls`, `network ls`, `network connect`

### ส่วนที่ 3: Systemd Services
- `systemctl status/start/stop/restart/enable/disable`
- `journalctl -f -u` (ดู log service), `--since`, `-n`
- `list-units --type=service --state=running`

### ส่วนที่ 4: Security
- **SSH Security**: เปลี่ยน port (2222), ปิด root login, บังคับ SSH Key, `MaxAuthTries`, `LoginGraceTime`
- **Fail2ban**: บล็อก IP brute force (bantime, findtime, maxretry)
- **Automatic Updates**: `unattended-upgrades` สำหรับ security patches
- **Password Policy**: `libpam-pwquality` (minlen, minclass, maxrepeat)

### ส่วนที่ 5: Backup Strategy
- **3-2-1 Rule**: 3 copies, 2 media types, 1 off-site
- **Backup Script**:
  1. หยุด database containers ก่อน
  2. Backup Docker volumes (`tar -czf`)
  3. Backup database โดยตรง (`pg_dumpall`)
  4. ลบ backup เก่ากว่า 30 วัน (`find -mtime +30 -delete`)
- **Cloud Backup**: `rclone sync` ไป Google Drive / Backblaze B2
- **Crontab**: รันทุกวัน 02:00 น.

### ส่วนที่ 6: Log Management
- **System Logs**: `journalctl -f`, `-u docker`, `--since`, `-p err`
- **Log Files**: `/var/log/syslog`, `/var/log/auth.log`
- **Logrotate**: หมุนเวียน log ไม่ให้เต็ม disk (daily, rotate 30, compress)

### ส่วนที่ 7: Performance Tuning
- **Kernel Parameters** (`/etc/sysctl.conf`):
  - `vm.swappiness = 10` (ใช้ RAM ก่อน Swap)
  - `fs.inotify.max_user_watches = 524288` (สำหรับ Node.js/apps)
  - Network tuning (rmem_max, wmem_max, tcp_rmem, tcp_wmem)
- **Docker Performance**: จำกัด resource (`--memory`, `--cpus`), `docker system prune`

### ส่วนที่ 8: Crontab
- รูปแบบ: `นาที ชั่วโมง วัน เดือน วันในสัปดาห์ คำสั่ง`
- ตัวอย่าง: backup ทุกวัน 02:00, prune ทุกอาทิตย์, health check ทุก 5 นาที

### ส่วนที่ 9: Environment Variables & Secrets
- **ใช้ .env file**: ไม่เขียน secret ใน code, `chmod 600 .env`, เพิ่มใน `.gitignore`
- **Docker Secrets** (สำหรับ Swarm): `docker secret create`

### ส่วนที่ 10: Git for Config Management
- เก็บ docker configs ใน git repo
- `.gitignore`: `*.env`, `**/data/`, `**/letsencrypt/`, `**/*.key`, `**/*.pem`

### ส่วนที่ 11: SSL Certificates
- **Let's Encrypt กับ Certbot**: `certbot --nginx`, renew อัตโนมัติทุก 90 วัน
- ตำแหน่งไฟล์: `/etc/letsencrypt/live/domain/fullchain.pem`, `privkey.pem`

### ส่วนที่ 12: Troubleshooting
- **Container ไม่ยอมเริ่ม**: ดู logs, exit code, port ชน, permission denied, volume ไม่มี
- **Disk เต็ม**: `df -h`, `du -sh`, `docker system prune -a`, `journalctl --vacuum-size`
- **RAM เต็ม**: `free -h`, `docker stats`, `ps aux --sort=-%mem`
- **Network ไม่ทำงาน**: `ping 8.8.8.8`, `ping google.com`, เปลี่ยน DNS
- **SSH เข้าไม่ได้**: ลอง local network, check sshd service, check firewall, check fail2ban

### ส่วนที่ 13: Monitoring
- **Health Check Script**:
  - CPU threshold 90%
  - RAM threshold 90%
  - Disk threshold 85%
  - แจ้งเตือนทาง Telegram

### ส่วนที่ 14: Update Strategy
- Security patches (OS): อัตโนมัติทุกวัน
- Docker images: ทุกเดือน หรือเมื่อมี security fix
- OS major version: ทุก 2 ปี
- Ubuntu LTS support: 5 ปี

### ส่วนที่ 15: Gotchas (สิ่งที่ไม่ควรทำ)
- ❌ อย่า run Docker ด้วย root user โดยตรง
- ❌ อย่า expose database port ออกนอก
- ❌ อย่าเปิด Port 22 ถ้าใช้ Tailscale แล้ว
- ❌ อย่าใช้ password เดิมสำหรับทุก service
- ❌ อย่าลบ volume โดยไม่ backup ก่อน
- ❌ อย่า update ทุกอย่างพร้อมกัน
- ❌ อย่าทำ `rm -rf` โดยไม่ระวัง
- ❌ อย่าเก็บ API Key ใน code หรือ git
- ❌ อย่าลืมตั้ง restart policy

### ส่วนที่ 16: Production Checklist
- OS & Security: Ubuntu 24.04 LTS, Static IP, SSH Key, UFW, Fail2ban, unattended-upgrades, Swap 8GB
- Docker: Docker + Compose, User in docker group, "homelab" network, /opt/docker structure, Portainer
- Network & Access: Tailscale, Cloudflare Tunnel, Domain, HTTPS
- Services: restart: unless-stopped, .env files, Volume paths
- Backup: Script tested, Crontab set, Restore tested
- Monitoring: Uptime Kuma, Telegram alerts, Health check script

## ข้อมูล / หลักฐาน ที่น่าสนใจ

### Backup Script Example
```bash
#!/usr/bin/bash
set -e
BACKUP_ROOT="/srv/nas/backup"
DATE=$(date +%Y-%m-%d_%H%M)

# 1. หยุด database containers
docker stop postgres 2>/dev/null || true

# 2. Backup Docker volumes
tar -czf "$BACKUP_ROOT/docker/docker-$DATE.tar.gz" /opt/docker/

# 3. เริ่ม containers
docker start postgres 2>/dev/null || true

# 4. Backup database โดยตรง
docker exec postgres pg_dumpall -U postgres > "$BACKUP_ROOT/db/postgres-$DATE.sql"
gzip "$BACKUP_ROOT/db/postgres-$DATE.sql"

# 5. ลบ backup เก่า
find "$BACKUP_ROOT" -name "*.tar.gz" -mtime +30 -delete
```

### SSH Security Config
```
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
LoginGraceTime 20
```

### Kernel Tuning
```
vm.swappiness = 10
fs.inotify.max_user_watches = 524288
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
```

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/docker-containers|Docker Containers]] — Container orchestration & management
- [[wiki/concepts/linux-system-administration|Linux System Administration]] — OS-level management
- [[wiki/concepts/server-security|Server Security]] — Security hardening
- [[wiki/concepts/backup-strategy|Backup Strategy]] — 3-2-1 backup rule
- [[wiki/concepts/systemd-services|Systemd Services]] — Service management
