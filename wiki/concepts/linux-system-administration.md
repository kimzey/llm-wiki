---
title: "Linux System Administration"
type: concept
tags: [linux, sysadmin, command-line, ubuntu]
sources: [wiki/sources/homeserver-admin-knowledge.md]
related: []
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

Linux System Administration คือศาสตร์การจัดการ Linux server ผ่าน command line — ครอบคลุม file management, permissions, processes, networking, monitoring และ troubleshooting

## อธิบาย

Linux (especially Ubuntu Server) เป็น OS หลักของ servers ทั่วโลก เพราะเสถียร, secure, และฟรี — sysadmin ต้องใช้ Terminal สั่งงานเป็นหลัก เรียนรู้ command line เป็นพื้นฐานสำคัญ

## ประเด็นสำคัญ

### 1. File & Directory Management

```bash
# Navigation
pwd                              # ดูว่าอยู่ที่ไหน
ls -la                           # ดูไฟล์ทุกอย่าง + permission
cd /opt/docker                   # เปลี่ยน directory

# Creation
mkdir folder                     # สร้าง directory
mkdir -p a/b/c                   # สร้าง path พร้อมกัน
touch file.txt                   # สร้างไฟล์ว่าง

# Copy, Move, Delete
cp file.txt backup.txt           # copy
cp -r folder/ backup/            # copy folder
mv file.txt /opt/                # move
mv old.txt new.txt               # rename
rm file.txt                      # delete
rm -rf folder/                   # delete folder (ระวัง!)

# View content
cat file.txt                     # แสดงทั้งหมด
less file.txt                    # แสดงทีละหน้า (q to exit)
head -20 file.txt                # 20 บรรทัดแรก
tail -f log.txt                  # real-time log
```

### 2. Text Editor — nano

```bash
nano /etc/hosts                  # เปิดไฟล์แก้ไข

# ใน nano:
# Ctrl+O  → Save
# Ctrl+X  → Exit
# Ctrl+W  → Search
# Ctrl+K  → Cut line
# Ctrl+U  → Paste line
```

### 3. Permissions (สิทธิ์ไฟล์)

```bash
ls -la
# → -rwxr-xr-- 1 admin docker 4096 Jan 1 script.sh
#    ││││││││││││  │
#    ││││││││││││  └─ group name
#    ││││││││││└──── owner name
#    ││││││││└────── file size
#    │││││└───────── others permission (--- = no access)
#    │││└─────────── group permission (r-x = read+execute)
#    ├──└──────────── owner permission (rwx = all)
#    └─────────────── file type (- = file, d = directory)

# เปลี่ยน permission
chmod 755 script.sh              # rwxr-xr-x
chmod +x script.sh               # เพิ่ม execute permission
chmod -R 755 /opt/docker         # recursive

# เปลี่ยนเจ้าของ
chown admin:admin file.txt
chown -R admin:admin /opt/docker
```

**Permission Numeric System:**
- `4` = read (r)
- `2` = write (w)
- `1` = execute (x)
- `7` = rwx (4+2+1)
- `6` = rw- (4+2)
- `5` = r-x (4+1)

### 4. Process Management

```bash
# ดู processes
ps aux                           # ทุก process
ps aux | grep docker             # filter
htop                             # interactive (ต้องติดตั้ง)

# หยุด process
kill 1234                        # graceful stop
kill -9 1234                     # force kill
killall nginx                    # kill by name

# Background
command &                        # run in background
nohup command &                  # run แม้ terminal ปิด
jobs                             # ดู background jobs
fg %1                            # นำกลับมา foreground
```

### 5. Pipe & Redirect (ทรงพลัง!)

```bash
# Pipe (|) — ส่ง output ไปคำสั่งถัดไป
docker ps | grep running         # filter
cat log.txt | grep ERROR         # หา ERROR
df -h | grep /dev/sda            # filter output

# Redirect
echo "hello" > file.txt          # เขียนทับ
echo "world" >> file.txt         # append
command 2>&1 | tee log.txt       # save + display

# Search
grep "ERROR" /var/log/syslog     # หา pattern
grep -r "pattern" /opt/          # recursive search
grep -i "error" log.txt          # case-insensitive
```

### 6. Network Commands

```bash
# ดู IP
ip a                             # ดู IP addresses
ip addr show

# Test connectivity
ping google.com                  # test internet
ping 192.168.1.1                 # test router

# ดู ports
ss -tlnp                         # ports ที่ listen อยู่
netstat -tlnp                    # แบบเก่า

# Routing & DNS
ip route                         # routing table
cat /etc/resolv.conf             # DNS server
nslookup google.com              # DNS lookup
dig yourdomain.com               # detailed DNS

# Connections
ss -tnp                          # connections ที่เชื่อมต่ออยู่
```

### 7. Disk & Memory

```bash
# Disk space
df -h                            # ทุก partition
df -h /opt/docker                # เฉพาะ path
du -sh /opt/docker/              # ขนาดรวม folder
du -sh /opt/docker/*             # ขนาดแต่ละ subfolder

# RAM
free -h                          # human-readable
cat /proc/meminfo                # detailed

# Disk I/O
iostat -x 2                      # ทุก 2 วินาที
```

### 8. Systemd Services

```bash
# Service management
sudo systemctl status docker
sudo systemctl start/stop/restart docker
sudo systemctl enable/disable docker      # auto-start on boot

# Logs
sudo journalctl -u docker -f              # real-time
sudo journalctl -u docker -n 100          # 100 lines
sudo journalctl -u docker --since "1h"    # 1 ชั่วโมงล่าสุด

# List services
sudo systemctl list-units --type=service --state=running
```

### 9. Crontab — Scheduled Tasks

```bash
crontab -e                       # edit crontab

# Format:
# นาที  ชั่วโมง  วัน  เดือน  วันในสัปดาห์  คำสั่ง
# 0-59  0-23    1-31  1-12   0-7(0=อาทิตย์)

# Examples:
0 2 * * *     /usr/local/bin/backup.sh           # ทุกวัน 02:00
0 3 * * 0     docker system prune -f             # ทุกอาทิตย์ 03:00
*/5 * * * *   /usr/local/bin/health-check.sh     # ทุก 5 นาที
```

## Troubleshooting Commands

### Disk Full
```bash
df -h                            # เช็คว่า disk เต็มจริงไหม
du -sh /opt/docker/* | sort -rh | head -20    # อะไรกินพื้นที่มากสุด
docker system prune -a           # ลบของ Docker ที่ไม่ใช้
sudo journalctl --vacuum-size=500M             # ลบ log เก่า
```

### RAM Full
```bash
free -h                          # เช็ค RAM
docker stats --no-stream | sort -k4 -rh        # containers กิน RAM
ps aux --sort=-%mem | head -20                 # processes กิน RAM
```

### Network Issues
```bash
ping 8.8.8.8                     # test internet
ping google.com                  # test DNS
ip route                         # check routing
sudo systemctl restart systemd-networkd        # restart network
```

## Best Practices

### File Operations
- **เช็ค path ก่อนลบ** — `rm -rf` ไม่มี undo!
- **ใช้ tab completion** — พิมพ์ path ผิดน้อยลง
- **backup ก่อนแก้** — `cp file.txt file.txt.bak`

### Security
- **อย่า run ด้วย root** ถ้าไม่จำเป็น — ใช้ `sudo`
- **Set permission ต่ำสุด** — เปิดเฉพาะที่จำเป็น
- **อย่าเก็บ password ใน plain text** — ใช้ .env หรือ secrets

### Monitoring
- **ตั้ง logrotate** — ไม่ให้ log เต็ม disk
- **ใช้ journalctl** — ดู log ระบบ centrally
- **ตั้ง alert** — แจ้งเตือนเมื่อ resource สูง

## Common Gotchas

❌ **`rm -rf /`** — ลบทุกอย่างใน system (ไม่มี trash)

❌ **`chmod 777`** — เปิด permission ทุกอย่าง (insecure)

❌ **`:wq` in nano** — นั่น vim command นะ

❌ **ลอง `kill -9` ก่อน `kill`** — force kill ไม่ graceful

## Quick Reference

| Task | Command |
|------|---------|
| Find file | `find /opt -name "*.yml"` |
| Find in files | `grep -r "pattern" /opt/` |
| File size | `du -sh file.txt` |
| Disk usage | `df -h` |
| Memory | `free -h` |
| Running processes | `ps aux` |
| Open ports | `ss -tlnp` |
| System log | `journalctl -f` |
| Service status | `systemctl status docker` |
| Test network | `ping 8.8.8.8` |

## แหล่งเรียนรู้เพิ่มเติม

- **linuxcommand.org** — ฟรี ครบมาก
- **tldr.sh** — ดูตัวอย่างคำสั่งด่วน
- **explainshell.com** — วิเคราะห์คำสั่งที่ไม่เข้าใจ
- **OverTheWire: Bandit** — เกมฝึก Linux security
