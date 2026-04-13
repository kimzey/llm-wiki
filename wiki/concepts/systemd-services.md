---
title: "Systemd Services"
type: concept
tags: [systemd, services, process-management, ubuntu]
sources: [wiki/sources/homeserver-admin-knowledge.md]
related: []
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

Systemd คือ init system และ service manager ของ Ubuntu — จัดการ background processes (services), handle dependencies, auto-start on boot, logging รวมศูนย์ — เป็นหัวใจของ Linux server administration

## อธิบาย

Systemd คือ "first process" (PID 1) ที่รันตอน boot — รับผิดชอบเริ่ม, หยุด, restart services ต่างๆ แทน traditional init scripts — มี unit files กำหนดว่า service ทำงานยังไง

## ประเด็นสำคัญ

### Basic Commands

```bash
# ดูสถานะ service
sudo systemctl status docker
sudo systemctl status cloudflared
sudo systemctl status tailscaled

# เริ่ม/หยุด/restart
sudo systemctl start docker
sudo systemctl stop docker
sudo systemctl restart docker

# ตั้งค่า auto-start on boot
sudo systemctl enable docker          # start เมื่อ boot
sudo systemctl disable docker         # ไม่ start เมื่อ boot

# ตรวจสอบ enabled/disabled
sudo systemctl is-enabled docker     # echo $? 0=enabled, 1=disabled
sudo systemctl is-active docker      # echo $? 0=running

# ดู services ทั้งหมดที่รันอยู่
sudo systemctl list-units --type=service --state=running

# ดู services ทั้งหมด (รวมที่หยุด)
sudo systemctl list-units --type=service --all
```

### Understanding Status Output

```
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: active (running) since Mon 2024-01-01 10:00:00; 1h ago
       Docs: https://docs.docker.com
   Main PID: 1234 (dockerd)
      Tasks: 152
     Memory: 450M
        CPU: 5.3s
     CGroup: /system.slice/docker.service
             └─1234 /usr/bin/dockerd
```

**Key fields:**
- **Loaded** — config โหลดแล้ว, enabled = auto-start
- **Active** — running หรือ not
- **Main PID** — process ID หลัก
- **Memory/CPU** — resource usage

### Viewing Logs — journalctl

Systemd รวม logs ไว้ใน journald — ดูด้วย `journalctl`

```bash
# Real-time log (เหมือน tail -f)
sudo journalctl -f                          # ทุก service
sudo journalctl -f -u docker               # เฉพาะ docker service

# ดู log ย้อนหลัง
sudo journalctl -u docker -n 100            # 100 บรรทัดล่าสุด
sudo journalctl -u docker --since "1 hour ago"
sudo journalctl -u docker --since "2024-01-01" --until "2024-01-02"

# Filter by priority
sudo journalctl -p err                     # errors เท่านั้น
sudo journalctl -p err -u docker           # errors ของ docker
# priorities: emerg(0), alert(1), crit(2), err(3), warning(4), notice(5), info(6), debug(7)

# Follow และ filter พร้อมกัน
sudo journalctl -f -u docker | grep ERROR

# Disk usage ของ logs
sudo journalctl --disk-usage
# vacuum (ลบ) log เก่า
sudo journalctl --vacuum-size=500M
sudo journalctl --vacuum-time=30d
```

### Creating Custom Service

ถ้าอยากให้ script หรือ app รันเป็น service:

```bash
sudo nano /etc/systemd/system/myapp.service
```

```ini
[Unit]
Description=My Custom Application
After=network.target                 # start หลัง network

[Service]
Type=simple
User=admin                          # run ด้วย user นี้
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/python3 /opt/myapp/app.py
Restart=always                      # restart เมื่อ crash
RestartSec=10                       # รอ 10 วินาทีก่อน restart
StandardOutput=journal              # log ไป journal
StandardError=journal

[Install]
WantedBy=multi-user.target          # start เมื่อ multi-user mode
```

```bash
# Reload systemd หลังแก้
sudo systemctl daemon-reload

# เริ่ม service
sudo systemctl start myapp
sudo systemctl enable myapp         # auto-start

# ดู log
sudo journalctl -f -u myapp
```

### Service Types

**Type=simple** (default)
```ini
[Service]
Type=simple
ExecStart=/usr/bin/myapp
# systemd ถือว่า service ready ทันทีหลัง fork
```

**Type=forking**
```ini
[Service]
Type=forking
ExecStart=/usr/bin/myapp --daemon
PIDFile=/var/run/myapp.pid
# app ต้อง daemonize เอง (background)
```

**Type=oneshot**
```ini
[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
# รันครั้งเดียวแล้วจบ
```

**Type=notify**
```ini
[Service]
Type=notify
ExecStart=/usr/bin/myapp
# app ส่ง signal บอกว่า ready (sd_notify)
```

### Restart Policies

```ini
[Service]
Restart=no                   # ไม่ restart เลย
Restart=on-success           # restart ถ้า exit 0
Restart=on-failure           # restart ถ้า exit != 0
Restart=on-abnormal          # crash, timeout, signal
Restart=always               # restart เสมอ (common สำหรับ long-running)
Restart=on-aborg             │

# Restart delay
RestartSec=10s               # รอ 10 วินาที
# หรือ exponential backoff
RestartSec=100ms             # 100ms → 200ms → 400ms → ...
```

### Docker Services

Docker services ควรเป็น **Type=notify**:

```ini
[Unit]
Description=Docker Application Container Engine
After=network-online.target docker.socket firewalld.service
Wants=network-online.target

[Service]
Type=notify
NotifyAccess=all
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
```

### Dependencies

```ini
[Unit]
Description=My App
After=network.target                # start หลัง network
Wants=database.service              # ชอบให้ database run แต่ไม่ required
Requires=redis.service              # required หากไม่มี redis ไม่ start
Before=nginx.service                # start ก่อน nginx
```

### Timer Units (Scheduled Tasks)

ทางเลือกหนึ่งของ crontab:

```bash
# /etc/systemd/system/backup.service
[Unit]
Description=Daily Backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh

# /etc/systemd/system/backup.timer
[Unit]
Description=Run backup daily

[Timer]
OnCalendar=daily                   # ทุกวัน
OnCalendar=02:00                   # เวลา 02:00
Persistent=true                    # run ถ้า miss

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl start backup.timer
sudo systemctl enable backup.timer
# เช็ค timers
sudo systemctl list-timers
```

### Resource Limits

```ini
[Service]
ExecStart=/usr/bin/myapp
# Memory limit
MemoryMax=512M
# CPU quota (50% of 1 core)
CPUQuota=50%
# Max file descriptors
LimitNOFILE=65536
# Max threads
LimitNPROC=4096
```

### Environment Variables

```ini
[Service]
ExecStart=/usr/bin/myapp
Environment="API_KEY=xxx"
Environment="LOG_LEVEL=debug"
EnvironmentFile=/opt/myapp/.env     # โหลดจาก file
```

### Troubleshooting

**Service ไม่ยอม start:**
```bash
# ดูว่า error อะไร
sudo systemctl status myapp
sudo journalctl -u myapp -n 50

# Check config syntax
sudo systemd-analyze verify myapp.service

# Check dependencies
sudo systemctl list-dependencies myapp
```

**Service start แล้ว crash ทันที:**
```bash
# Check resource limits
sudo journalctl -u myapp | grep -i "out of"

# Check permissions
ls -la /opt/myapp

# Test manual run
sudo -u admin /usr/bin/myapp
```

**Service ไม่ auto-start on boot:**
```bash
# Check enabled
sudo systemctl is-enabled myapp

# Check required services
sudo systemctl show myapp -p Requires,After,Before
```

### Systemd Targets (Runlevels)

Systemd ใช้ "targets" แทน runlevels:

```bash
# Default target (เหมือน runlevel)
sudo systemctl get-default           # multi-user.target หรือ graphical.target

# Switch target
sudo systemctl isolate graphical.target

# Set default
sudo systemctl set-default graphical.target
```

**Common targets:**
- `multi-user.target` — non-graphical (server default)
- `graphical.target` — GUI (desktop default)
- `rescue.target` — single user mode (maintenance)
- `emergency.target` — minimal emergency shell

## Best Practices

### 1. Use Restart=always สำหรับ Long-Running Services
```ini
[Service]
Restart=always
RestartSec=10
```

### 2. Log ไป Journal (เพื่อรวม logs)
```ini
[Service]
StandardOutput=journal
StandardError=journal
```

### 3. Run ด้วย Non-Root User
```ini
[Service]
User=admin
Group=admin
```

### 4. Set Resource Limits
```ini
[Service]
MemoryMax=1G
CPUQuota=200%
```

### 5. Use Type=notify สำหรับ Modern Apps
```ini
[Service]
Type=notify
NotifyAccess=all
```

## Comparison: Systemd vs Crontab

| Feature | Systemd Timer | Crontab |
|---------|---------------|---------|
| Logging | ✅ journalctl integrated | ❌ manual |
| Email on failure | ✅ built-in | ⚠️ manual |
| Timezone handling | ✅ aware | ❌ system-only |
| Boot-time run | ✅ if missed | ❌ no |
| Resource control | ✅ limits | ❌ no |
| Ease of use | ⚠️ verbose | ✅ simple |

**Recommendation:** Use systemd timers สำหรับ critical jobs (backup, monitoring), crontab สำหรับ simple recurring tasks

## Quick Reference

```bash
# Status
systemctl status name
systemctl is-active name
systemctl is-enabled name

# Control
systemctl start/stop/restart name
systemctl enable/disable name
systemctl reload name

# Logs
journalctl -u name
journalctl -u name -f
journalctl -u name -n 100

# List
systemctl list-units --type=service
systemctl list-timers

# Custom service
systemctl edit --full name    # edit unit file
systemctl daemon-reload       # reload หลังแก้
```

## แหล่งเรียนรู้เพิ่มเติม

- **freedesktop.org/software/systemd/man/** — official documentation
- **coreos.com/systemd** — Systemd essentials
- **linux.die.net/man/5/systemd.service** — service config reference
- **unix.stackexchange.com** — Q&A ครบมาก
