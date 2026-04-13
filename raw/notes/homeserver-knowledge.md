# 📚 สิ่งที่ต้องรู้และควรรู้ — Home Server Admin
### ครอบคลุมทุกเรื่องที่จะเจอจริงๆ ในการดูแล Server

---

## ส่วนที่ 1: Linux Command Line พื้นฐาน

Linux คือ OS ของ Server คุณ ต้องใช้ Terminal สั่งงานเป็นหลัก

### 1.1 การจัดการไฟล์และ Directory

```bash
# ดูว่าอยู่ที่ไหน
pwd
# → /home/admin

# ดูไฟล์ใน directory ปัจจุบัน
ls
ls -la          # แสดงทุกไฟล์รวม hidden + permission + ขนาด

# เปลี่ยน directory
cd /opt/docker
cd ~            # กลับ home
cd ..           # ขึ้นไป 1 ระดับ
cd -            # กลับ directory ก่อนหน้า

# สร้าง directory
mkdir myfolder
mkdir -p a/b/c  # สร้างทั้ง path พร้อมกัน

# คัดลอก ย้าย ลบ
cp file.txt backup.txt
cp -r folder/ backup-folder/   # คัดลอก folder ทั้งหมด
mv file.txt /opt/docker/       # ย้ายไฟล์
mv old-name.txt new-name.txt   # เปลี่ยนชื่อ
rm file.txt
rm -rf folder/  # ลบ folder ทั้งหมด (ระวัง! ไม่มี trash)

# ดูเนื้อหาไฟล์
cat file.txt            # แสดงทั้งหมด
less file.txt           # แสดงทีละหน้า (q เพื่อออก)
head -20 file.txt       # แสดง 20 บรรทัดแรก
tail -20 file.txt       # แสดง 20 บรรทัดสุดท้าย
tail -f log.txt         # แสดง real-time (ดู log)
```

### 1.2 การแก้ไขไฟล์ด้วย nano (ง่ายที่สุด)

```bash
nano /etc/hosts         # เปิดไฟล์แก้ไข

# ใน nano:
# Ctrl+O  → Save
# Ctrl+X  → ออก
# Ctrl+W  → ค้นหา
# Ctrl+K  → ตัดบรรทัด
# Ctrl+U  → วางบรรทัด
```

### 1.3 Permission (สิทธิ์ไฟล์)

```bash
# ดู permission
ls -la
# → -rwxr-xr-- 1 admin docker 4096 Jan 1 script.sh
#    ↑↑↑↑↑↑↑↑↑
#    │││││││││
#    ││││││├──  --- = others (คนอื่น)
#    │││├──     --- = group (กลุ่ม)
#    ├──         rwx = owner (เจ้าของ)
#    │
#    r=read, w=write, x=execute

# เปลี่ยน permission
chmod 755 script.sh     # rwxr-xr-x
chmod +x script.sh      # เพิ่ม execute permission
chmod -R 755 /opt/docker  # เปลี่ยนทั้ง folder

# เปลี่ยนเจ้าของ
chown admin:admin file.txt
chown -R admin:admin /opt/docker  # ทั้ง folder
```

### 1.4 Process Management

```bash
# ดู process ที่รันอยู่
ps aux
ps aux | grep docker    # กรองเฉพาะที่มีคำว่า docker

# ดูแบบ interactive (กด q ออก)
htop                    # ต้องติดตั้งก่อน: apt install htop

# หยุด process
kill 1234               # หยุดด้วย PID
kill -9 1234            # บังคับหยุด (force)
killall nginx           # หยุดด้วยชื่อ

# รัน process ใน background
command &               # รันใน background
nohup command &         # รันแม้ terminal ปิด
```

### 1.5 Pipe และ Redirect (ทรงพลังมาก)

```bash
# Pipe (|) ส่ง output ไปให้คำสั่งถัดไป
docker ps | grep running          # กรองเฉพาะที่รันอยู่
cat log.txt | grep ERROR          # หา ERROR ใน log
df -h | grep /dev/sda             # ดู disk เฉพาะ partition

# Redirect (> และ >>)
echo "hello" > file.txt           # เขียนทับ
echo "world" >> file.txt          # เพิ่มต่อท้าย
command 2>&1 | tee output.log     # save output ไปไฟล์ด้วย

# ค้นหาใน output
grep "ERROR" /var/log/syslog      # หา pattern
grep -r "pattern" /opt/docker/    # ค้นใน folder ทั้งหมด
grep -i "error" log.txt           # ไม่สนตัวพิมพ์เล็กใหญ่
```

### 1.6 Network Commands

```bash
# ดู IP ของเครื่อง
ip a
ip addr show

# ทดสอบการเชื่อมต่อ
ping google.com
ping 192.168.1.1        # ping router

# ดู port ที่เปิดอยู่
ss -tlnp                # แสดง port ที่ listen อยู่
netstat -tlnp           # แบบเก่า

# ดู route
ip route

# ดู DNS
cat /etc/resolv.conf
nslookup google.com
dig yourdomain.com

# ดู connection ที่เชื่อมต่ออยู่
ss -tnp
```

### 1.7 Disk และ Memory

```bash
# ดู disk space
df -h                   # ดูทุก partition
df -h /opt/docker       # ดูเฉพาะ path นั้น
du -sh /opt/docker/     # ขนาดรวมของ folder
du -sh /opt/docker/*    # ขนาดแต่ละ sub-folder

# ดู RAM
free -h
cat /proc/meminfo

# ดู disk I/O
iostat -x 2             # ดู disk usage ทุก 2 วินาที
```

---

## ส่วนที่ 2: Docker Commands ที่ใช้บ่อย

### 2.1 Container Management

```bash
# ดู container ที่รันอยู่
docker ps
docker ps -a            # รวม container ที่หยุดแล้ว

# เริ่ม/หยุด/restart
docker start homeassistant
docker stop homeassistant
docker restart homeassistant

# ดู log
docker logs homeassistant
docker logs -f homeassistant        # real-time
docker logs --tail 50 homeassistant # 50 บรรทัดล่าสุด
docker logs --since 1h homeassistant # 1 ชั่วโมงล่าสุด

# เข้าไปใน container
docker exec -it homeassistant bash
docker exec -it homeassistant sh    # ถ้าไม่มี bash

# ดู resource usage
docker stats
docker stats homeassistant

# ดูข้อมูล container
docker inspect homeassistant
```

### 2.2 Docker Compose

```bash
# รัน services จาก docker-compose.yml
cd /opt/docker/homeassistant
docker compose up -d        # รัน background
docker compose up           # รัน แสดง log

# หยุด services
docker compose down
docker compose down -v      # ลบ volume ด้วย (ระวัง!)

# อัปเดต image ใหม่
docker compose pull         # ดึง image ใหม่
docker compose up -d        # restart ด้วย image ใหม่

# ดู log ทุก service
docker compose logs -f
docker compose logs -f homeassistant  # เฉพาะ service

# restart service เดียว
docker compose restart homeassistant
```

### 2.3 Image Management

```bash
# ดู images ที่มี
docker images

# ดึง image ใหม่
docker pull ubuntu:24.04

# ลบ image
docker rmi image_name

# ทำความสะอาด (ลบของที่ไม่ใช้)
docker system prune          # ลบ container/network/image ที่ไม่ใช้
docker system prune -a       # ลบทุกอย่างที่ไม่ใช้ (รวม image)
docker volume prune          # ลบ volume ที่ไม่ใช้
```

### 2.4 Volume และ Network

```bash
# ดู volume
docker volume ls
docker volume inspect portainer_data

# ดู network
docker network ls
docker network inspect homelab

# เพิ่ม container เข้า network
docker network connect homelab container_name
```

---

## ส่วนที่ 3: System Service และ Systemd

Ubuntu ใช้ systemd จัดการ services ของระบบ

```bash
# ดูสถานะ service
sudo systemctl status docker
sudo systemctl status cloudflared
sudo systemctl status tailscaled

# เริ่ม/หยุด/restart service
sudo systemctl start docker
sudo systemctl stop docker
sudo systemctl restart docker

# ตั้งให้ start อัตโนมัติตอน boot
sudo systemctl enable docker
sudo systemctl disable docker

# ดู log ของ service
sudo journalctl -u docker -f        # real-time
sudo journalctl -u docker -n 100    # 100 บรรทัดล่าสุด
sudo journalctl -u docker --since "1 hour ago"

# ดู services ทั้งหมดที่รันอยู่
sudo systemctl list-units --type=service --state=running
```

---

## ส่วนที่ 4: Security — สิ่งที่ต้องทำ

### 4.1 SSH Security

```bash
# แก้ไข SSH config
sudo nano /etc/ssh/sshd_config
```

```
# ค่าที่ควรตั้ง:
Port 2222                    # เปลี่ยนจาก 22 (ลดการสแกน)
PermitRootLogin no           # ห้าม login เป็น root
PasswordAuthentication no    # บังคับใช้ SSH Key เท่านั้น
PubkeyAuthentication yes     # ใช้ Key ได้
MaxAuthTries 3               # ลองผิดได้ 3 ครั้ง
LoginGraceTime 20            # timeout 20 วินาที
```

```bash
# Restart SSH หลังแก้
sudo systemctl restart sshd

# อย่าลืม! ต้องเพิ่ม port ใน UFW ก่อน
sudo ufw allow 2222/tcp
```

### 4.2 Fail2ban — บล็อก IP ที่ brute force

```bash
# ดูสถานะ
sudo fail2ban-client status
sudo fail2ban-client status sshd

# ดู IP ที่โดน ban
sudo fail2ban-client status sshd
# ตัวอย่าง output:
# Banned IP list: 185.224.128.17 91.92.251.103

# unban IP (ถ้า ban ตัวเอง)
sudo fail2ban-client unban 192.168.1.50
```

```ini
# /etc/fail2ban/jail.local — config หลัก
[DEFAULT]
bantime  = 3600      # ban 1 ชั่วโมง
findtime = 600       # ดูใน 10 นาที
maxretry = 5         # ลองผิด 5 ครั้ง

[sshd]
enabled = true
port    = 2222       # ถ้าเปลี่ยน port แล้ว
```

### 4.3 Automatic Security Updates

```bash
# ติดตั้ง unattended-upgrades
sudo apt install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
# เลือก Yes → จะ update security patches อัตโนมัติ

# ดูสถานะ
sudo systemctl status unattended-upgrades
```

### 4.4 Password Policy

```bash
# ติดตั้ง password quality checker
sudo apt install libpam-pwquality

# แก้ไข config
sudo nano /etc/security/pwquality.conf
```

```
minlen = 12          # อย่างน้อย 12 ตัว
minclass = 3         # ต้องมีอย่างน้อย 3 ประเภท (ตัวใหญ่/เล็ก/ตัวเลข)
maxrepeat = 3        # ห้ามซ้ำกัน 3 ตัวติดกัน
```

---

## ส่วนที่ 5: Backup Strategy

### 5.1 3-2-1 Rule (มาตรฐานสากล)

```
3 copies ของข้อมูล
  ├── 1. ข้อมูลหลักในเครื่อง NUC
  ├── 2. Backup ใน HDD External
  └── 3. Backup บน Cloud (Google Drive / Backblaze)

2 media ต่างประเภทกัน
  ├── SSD ใน NUC
  └── HDD External

1 copy อยู่ off-site (นอกบ้าน)
  └── Cloud storage
```

### 5.2 Script Backup อัตโนมัติ

```bash
# /usr/local/bin/backup.sh
#!/bin/bash
set -e

BACKUP_ROOT="/srv/nas/backup"
DATE=$(date +%Y-%m-%d_%H%M)
LOG="/var/log/backup.log"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG"; }

log "=== เริ่ม Backup ==="

# 1. หยุด container ที่มี database ก่อน (ป้องกัน data corruption)
log "หยุด database containers..."
docker stop postgres 2>/dev/null || true

# 2. Backup Docker volumes
log "Backup /opt/docker..."
mkdir -p "$BACKUP_ROOT/docker"
tar -czf "$BACKUP_ROOT/docker/docker-$DATE.tar.gz" \
  --exclude='/opt/docker/netdata' \
  /opt/docker/

# 3. เริ่ม container กลับ
log "เริ่ม database containers..."
docker start postgres 2>/dev/null || true

# 4. Backup database โดยตรง (ดีกว่า backup ไฟล์)
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

```bash
# ตั้ง crontab
sudo crontab -e
# เพิ่มบรรทัดนี้:
# 0 2 * * * /usr/local/bin/backup.sh
# (รันทุกวัน 02:00 น.)
```

### 5.3 Backup ขึ้น Cloud ด้วย rclone

```bash
# ติดตั้ง rclone
curl https://rclone.org/install.sh | sudo bash

# ตั้งค่า (interactive)
rclone config
# เลือก Google Drive หรือ Backblaze B2

# ทดสอบ sync
rclone sync /srv/nas/backup gdrive:homeserver-backup

# เพิ่มใน backup script
rclone sync /srv/nas/backup gdrive:homeserver-backup \
  --log-file=/var/log/rclone.log \
  --log-level INFO
```

---

## ส่วนที่ 6: Log Management

### 6.1 ดู Log ระบบ

```bash
# Log ระบบหลัก
sudo journalctl -f                          # real-time ทุก log
sudo journalctl -f -u docker               # เฉพาะ docker
sudo journalctl --since "2024-01-01" --until "2024-01-02"
sudo journalctl -p err                     # เฉพาะ error

# Log ไฟล์เก่า
sudo cat /var/log/syslog
sudo tail -f /var/log/auth.log             # ดู login attempts
sudo grep "Failed password" /var/log/auth.log  # ดูคนที่พยายาม login
```

### 6.2 Logrotate — หมุนเวียน log ไม่ให้เต็ม disk

```bash
# ดู config
cat /etc/logrotate.conf

# สร้าง config สำหรับ custom log
sudo nano /etc/logrotate.d/homeserver
```

```
/var/log/backup.log {
    daily
    rotate 30
    compress
    missingok
    notifempty
    create 0644 root root
}
```

---

## ส่วนที่ 7: Performance Tuning

### 7.1 Kernel Parameters

```bash
sudo nano /etc/sysctl.conf
```

```
# ลด swappiness (ใช้ RAM ก่อน Swap)
vm.swappiness = 10

# เพิ่ม file watchers (สำหรับ Node.js/apps)
fs.inotify.max_user_watches = 524288

# Network performance
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
```

```bash
sudo sysctl -p   # apply ทันที
```

### 7.2 Docker Performance

```bash
# จำกัด resource ของ container ที่ไม่สำคัญ
docker update --memory="512m" --cpus="0.5" netdata

# ดู resource usage ทุก container
docker stats --no-stream

# ทำความสะอาด Docker เป็นประจำ
# (เพิ่มใน crontab ทุกอาทิตย์)
docker system prune -f
```

---

## ส่วนที่ 8: Crontab — Scheduled Tasks

```bash
# แก้ไข crontab
crontab -e

# รูปแบบ:
# นาที  ชั่วโมง  วัน  เดือน  วันในสัปดาห์  คำสั่ง
# 0-59  0-23    1-31  1-12   0-7(0=อาทิตย์)

# ตัวอย่าง:
0 2 * * *     /usr/local/bin/backup.sh           # ทุกวัน 02:00
0 3 * * 0     docker system prune -f             # ทุกอาทิตย์ 03:00
*/5 * * * *   /usr/local/bin/health-check.sh     # ทุก 5 นาที
0 0 1 * *     certbot renew                      # ทุกเดือน
30 6 * * *    docker compose -f /opt/docker/homeassistant/docker-compose.yml pull  # อัปเดต image ทุกเช้า
```

---

## ส่วนที่ 9: Environment Variables และ Secrets

### 9.1 ไม่ควรเขียน Secret ใน Code!

```bash
# ผิด ❌
ANTHROPIC_API_KEY = "sk-ant-xxx"   # อย่าเขียนใน code!

# ถูก ✅ — ใช้ .env file
# สร้าง /opt/docker/agent/.env
ANTHROPIC_API_KEY=sk-ant-xxx
TELEGRAM_TOKEN=123456:ABC-xxx
MY_TELEGRAM_ID=987654321

# ใน docker-compose.yml
services:
  agent:
    env_file:
      - .env           # โหลด .env อัตโนมัติ
```

```bash
# ป้องกันไม่ให้ .env หลุด
chmod 600 /opt/docker/agent/.env   # เฉพาะเจ้าของอ่านได้

# เพิ่มใน .gitignore ถ้าใช้ git
echo ".env" >> .gitignore
echo "*.env" >> .gitignore
```

### 9.2 Docker Secrets (สำหรับ Swarm — production)

```bash
# สร้าง secret
echo "sk-ant-xxx" | docker secret create anthropic_key -

# ใช้ใน docker-compose.yml
secrets:
  anthropic_key:
    external: true
```

---

## ส่วนที่ 10: Git — จัดการ Config ด้วย Version Control

```bash
# ติดตั้ง git
sudo apt install git

# ตั้งค่า
git config --global user.name "Your Name"
git config --global user.email "your@email.com"

# สร้าง repo สำหรับเก็บ docker configs
mkdir /opt/homelab-config && cd /opt/homelab-config
git init

# Copy configs มา (ไม่เอา .env)
cp /opt/docker/homeassistant/docker-compose.yml ./homeassistant/
cp /opt/docker/n8n/docker-compose.yml ./n8n/

# ทำ .gitignore
cat > .gitignore << 'EOF'
*.env
.env*
**/data/
**/letsencrypt/
**/*.key
**/*.pem
EOF

git add .
git commit -m "initial homelab config"

# Push ขึ้น GitHub (private repo)
git remote add origin git@github.com:you/homelab.git
git push -u origin main
```

---

## ส่วนที่ 11: SSL Certificate จัดการเอง (ถ้าไม่ใช้ Cloudflare)

### Let's Encrypt กับ Certbot

```bash
# ติดตั้ง certbot
sudo apt install certbot python3-certbot-nginx

# ขอ certificate สำหรับ domain
sudo certbot --nginx -d yourdomain.com -d ha.yourdomain.com

# ต่ออายุอัตโนมัติ (Let's Encrypt หมดอายุทุก 90 วัน)
sudo certbot renew --dry-run   # ทดสอบก่อน
# certbot จะตั้ง cron job ให้อัตโนมัติ

# ดู certificate ที่มี
sudo certbot certificates

# ตำแหน่ง certificate ไฟล์
# /etc/letsencrypt/live/yourdomain.com/fullchain.pem
# /etc/letsencrypt/live/yourdomain.com/privkey.pem
```

---

## ส่วนที่ 12: Troubleshooting — แก้ปัญหาที่เจอบ่อย

### 12.1 Container ไม่ยอมเริ่ม

```bash
# ดูว่า error อะไร
docker logs container_name

# ดู exit code
docker ps -a
# STATUS: Exited (1) = error
# STATUS: Exited (0) = ปกติ

# ปัญหาที่เจอบ่อย:
# 1. Port ชนกัน
ss -tlnp | grep :8123      # ดูว่ามีอะไรใช้ port นี้อยู่

# 2. Permission denied
ls -la /opt/docker/homeassistant/
chown -R 1000:1000 /opt/docker/homeassistant/data/

# 3. Volume ไม่มี
mkdir -p /opt/docker/homeassistant/data
```

### 12.2 Disk เต็ม

```bash
# ดูว่าอะไรกิน disk มากสุด
df -h
du -sh /opt/docker/*   | sort -rh | head -20
du -sh /var/log/*      | sort -rh | head -20

# ทำความสะอาด Docker
docker system prune -a    # ลบ image/container/network ที่ไม่ใช้
docker volume prune       # ลบ volume ที่ไม่ใช้

# ดู log ที่ใหญ่
find /var/log -name "*.log" -size +100M
# ลบ log เก่า
sudo journalctl --vacuum-size=500M
```

### 12.3 RAM เต็ม

```bash
# ดูว่าอะไรกิน RAM มากสุด
free -h
docker stats --no-stream | sort -k4 -rh

# ดู process
ps aux --sort=-%mem | head -20

# หยุด container ที่ไม่ใช้ชั่วคราว
docker stop netdata       # monitoring ใช้ RAM เยอะ
```

### 12.4 Network ไม่ทำงาน

```bash
# ทดสอบขั้นตอน
ping 8.8.8.8              # ทดสอบ internet
ping google.com           # ทดสอบ DNS
ping 192.168.1.1          # ทดสอบ router

# ถ้า ping 8.8.8.8 ได้แต่ ping google.com ไม่ได้ = DNS มีปัญหา
# แก้โดยเปลี่ยน DNS
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf

# ดู routing
ip route
route -n

# Restart network
sudo systemctl restart systemd-networkd
sudo netplan apply
```

### 12.5 SSH เข้าไม่ได้

```bash
# ลองจาก local network ก่อน (ไม่ผ่าน Tailscale)
ssh admin@192.168.1.100

# ถ้าเข้าไม่ได้เลย ต้องต่อจอกับคีย์บอร์ดกับ NUC โดยตรง

# เช็ค SSH service
sudo systemctl status sshd
sudo journalctl -u sshd -n 50

# เช็ค firewall
sudo ufw status
sudo ufw allow ssh

# เช็ค fail2ban
sudo fail2ban-client status sshd
# ถ้า IP ตัวเองโดน ban:
sudo fail2ban-client unban YOUR_IP
```

---

## ส่วนที่ 13: Monitoring แบบ Manual

### 13.1 Script ตรวจสุขภาพ Server

```bash
# /usr/local/bin/health-check.sh
#!/bin/bash

# ตัวแปร
TELEGRAM_TOKEN="your-token"
CHAT_ID="your-chat-id"
THRESHOLD_CPU=90      # แจ้งเตือนถ้า CPU > 90%
THRESHOLD_RAM=90      # แจ้งเตือนถ้า RAM > 90%
THRESHOLD_DISK=85     # แจ้งเตือนถ้า Disk > 85%

send_alert() {
  curl -s -X POST "https://api.telegram.org/bot$TELEGRAM_TOKEN/sendMessage" \
    -d chat_id="$CHAT_ID" \
    -d text="⚠️ NUC Alert: $1" > /dev/null
}

# เช็ค CPU
CPU=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d. -f1)
[ "$CPU" -gt "$THRESHOLD_CPU" ] && send_alert "CPU สูง: ${CPU}%"

# เช็ค RAM
RAM=$(free | grep Mem | awk '{printf "%.0f", $3/$2*100}')
[ "$RAM" -gt "$THRESHOLD_RAM" ] && send_alert "RAM สูง: ${RAM}%"

# เช็ค Disk
DISK=$(df / | tail -1 | awk '{print $5}' | tr -d '%')
[ "$DISK" -gt "$THRESHOLD_DISK" ] && send_alert "Disk เต็ม: ${DISK}%"

# เช็ค container ที่หยุด
STOPPED=$(docker ps -a --filter "status=exited" --format "{{.Names}}" | tr '\n' ', ')
[ -n "$STOPPED" ] && send_alert "Container หยุด: $STOPPED"
```

---

## ส่วนที่ 14: Update Strategy

### อะไรควร Update บ่อยแค่ไหน?

```
Security patches (OS)     → อัตโนมัติทุกวัน (unattended-upgrades)
Docker images             → ทุกเดือน หรือเมื่อมี security fix
OS major version          → ทุก 2 ปี (ไม่รีบ)
Ubuntu LTS support        → 5 ปี (24.04 ถึงปี 2029)
```

```bash
# อัปเดต Docker images ปลอดภัย
# 1. Pull image ใหม่
docker compose pull

# 2. ดูว่ามีอะไรเปลี่ยน
docker compose diff    # ถ้ารองรับ

# 3. Stop → Update → Start
docker compose down
docker compose up -d

# 4. ดู log หลัง update
docker compose logs -f
```

---

## ส่วนที่ 15: สิ่งที่ไม่ควรทำ (Gotchas)

```
❌ อย่า run Docker ด้วย root user โดยตรง
   → ใช้ user ธรรมดาที่อยู่ใน group docker แทน

❌ อย่า expose database port ออกนอก
   → postgres, mysql ต้องอยู่ใน Docker network เท่านั้น

❌ อย่าเปิด Port 22 ถ้าใช้ Tailscale แล้ว
   → ใช้ Tailscale SSH แทน ปลอดภัยกว่ามาก

❌ อย่าใช้ password เดิมสำหรับทุก service
   → ใช้ Vaultwarden (self-hosted Bitwarden) จัดการ

❌ อย่าลบ volume โดยไม่ backup ก่อน
   → docker compose down -v จะลบข้อมูลทั้งหมด

❌ อย่า update ทุกอย่างพร้อมกัน
   → update ทีละ service แล้วดู log ก่อน

❌ อย่าทำ rm -rf โดยไม่ระวัง
   → ไม่มี undo, ไม่มี trash

❌ อย่าเก็บ API Key ใน code หรือ git
   → ใช้ .env file และ .gitignore

❌ อย่าลืมตั้ง restart policy
   → ทุก container ควรมี restart: unless-stopped
```

---

## ส่วนที่ 16: Checklist ก่อนใช้งาน Production

```
OS & Security
☐ Ubuntu Server 24.04 LTS ติดตั้งแล้ว
☐ Update ทั้งหมดแล้ว
☐ Static IP ตั้งแล้ว
☐ SSH Key ตั้งแล้ว (Password auth ปิด)
☐ UFW Firewall เปิดแล้ว
☐ Fail2ban ทำงานอยู่
☐ unattended-upgrades เปิดแล้ว
☐ Swap 8GB ตั้งแล้ว

Docker
☐ Docker + Docker Compose ติดตั้งแล้ว
☐ User อยู่ใน docker group
☐ Docker network "homelab" สร้างแล้ว
☐ Folder structure (/opt/docker) สร้างแล้ว
☐ Portainer ทำงานอยู่

Network & Access
☐ Tailscale ติดตั้งและเชื่อมต่อแล้ว
☐ Cloudflare Tunnel ทำงานอยู่
☐ Domain ชี้ถูกต้องทุก subdomain
☐ HTTPS ทำงานทุก domain

Services
☐ ทุก container มี restart: unless-stopped
☐ ทุก container ใช้ .env file สำหรับ secrets
☐ Volume paths ถูกต้อง

Backup
☐ Backup script สร้างแล้วและทดสอบแล้ว
☐ Crontab ตั้งแล้ว
☐ ทดสอบ restore จาก backup แล้ว

Monitoring
☐ Uptime Kuma ทำงานอยู่
☐ Telegram alerts ตั้งแล้ว
☐ Health check script ทำงานอยู่
```

---

## ส่วนที่ 17: แหล่งเรียนรู้เพิ่มเติม

```
Linux Command Line
→ linuxcommand.org (ฟรี ครบมาก)
→ tldr.sh (ดูตัวอย่างคำสั่งด่วน)
→ explainshell.com (วิเคราะห์คำสั่งที่ไม่เข้าใจ)

Docker
→ docs.docker.com/get-started (official)
→ youtube: TechWorld with Nana (อธิบายดีมาก)

Self-Hosted
→ reddit.com/r/selfhosted (community ใหญ่มาก)
→ awesome-selfhosted.net (รายการ apps ทั้งหมด)
→ noted.lol (blog เกี่ยวกับ homelab)

Security
→ haveibeenpwned.com (เช็คว่า email โดน leak ไหม)
→ shodan.io (เช็คว่า IP บ้านเปิด port อะไรบ้าง)

Monitoring
→ github.com/louislam/uptime-kuma
→ netdata.cloud
```

---

*เอกสารนี้เป็นส่วนหนึ่งของ Home Server Setup Plan*
*Intel NUC7i5BNH · Ubuntu 24.04 LTS · Docker*
