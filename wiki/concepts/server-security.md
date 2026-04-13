---
title: "Server Security"
type: concept
tags: [security, hardening, ssh, firewall, fail2ban, ufw, port-forwarding, vpn, cloudflare]
sources: [wiki/sources/homeserver-admin-knowledge.md, network-fundamentals.md]
related:
  - wiki/concepts/vpn-tailscale.md
  - wiki/concepts/cloudflare-tunnel.md
  - wiki/concepts/port-networking.md
  - wiki/concepts/ssl-tls.md
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

Server Security Hardening คือกระบวนการเสริมความปลอดภัย server ทีละชั้น — SSH hardening, Firewall, Intrusion Prevention, Automatic Updates, Password Policies — เพื่อป้องกันการโจมตีทั่วไป

## อธิบาย

Default configuration ของ Linux มักไม่ปลอดภัยพอสำหรับการ expose ออก internet — hardening คือการลด attack surface และเพิ่ม layers ของ protection เพื่อให้ attacker ยากต่อการบุก

## ประเด็นสำคัญ

### Layer 1: SSH Hardening

SSH คือประตูหลักเข้า server — ถ้าโดนบุกตรงนี้ เกมจบ

```bash
sudo nano /etc/ssh/sshd_config
```

**Config ที่ควรตั้ง:**
```ini
# เปลี่ยน port — ลด automated scanning
Port 2222                        # default 22

# ปิด root login
PermitRootLogin no               # ห้าม login เป็น root

# บังคับใช้ SSH Key หมายเลย
PasswordAuthentication no        # ปิด password auth
PubkeyAuthentication yes         # เปิด key auth

# จำกัด attempts
MaxAuthTries 3                   # ลองผิดได้ 3 ครั้ง
LoginGraceTime 20                # timeout 20 วินาที

# จำกัด users (ถ้าต้องการ)
AllowUsers admin kim             # เฉพาะ users นี้ login ได้
# AllowGroups ssh-users
```

**Generate SSH Key:**
```bash
# ที่เครื่อง client
ssh-keygen -t ed25519 -C "your@email.com"

# Copy key ไป server
ssh-copy-id -i ~/.ssh/id_ed25519.pub admin@server_ip

# ทดสอบ
ssh -p 2222 admin@server_ip
```

**Restart SSH:**
```bash
sudo systemctl restart sshd

# อย่าลืมเพิ่ม port ใน firewall ก่อน!
sudo ufw allow 2222/tcp
```

### Layer 2: Firewall — UFW

Uncomplicated Firewall (UFW) คือ firewall frontend ที่ง่ายที่สุดบน Ubuntu

```bash
# เปิด UFW
sudo ufw enable

# Default policies
sudo ufw default deny incoming    # block ทุก incoming
sudo ufw default allow outgoing   # allow ทุก outgoing

# Allow ports ที่จำเป็น
sudo ufw allow 2222/tcp           # SSH (ถ้าเปลี่ยน port)
sudo ufw allow 80/tcp             # HTTP
sudo ufw allow 443/tcp            # HTTPS

# ดู status
sudo ufw status numbered
```

**Tailscale Alternative:**
ถ้าใช้ Tailscale VPN — **อย่าเปิด Port 22/2222 เลย** ใช้ Tailscale SSH แทน ปลอดภัยกว่ามาก

### Layer 3: Fail2ban — Intrusion Prevention

Fail2ban คอยดู log และ auto-ban IP ที่พยายาม brute force

```bash
# ติดตั้ง
sudo apt install fail2ban

# Config หลัก
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime  = 3600                  # ban 1 ชั่วโมง
findtime = 600                   # ดูใน 10 นาที
maxretry = 5                     # ลองผิด 5 ครั้ง

[sshd]
enabled = true
port    = 2222                   # ถ้าเปลี่ยน SSH port
```

**Commands:**
```bash
# ดูสถานะ
sudo fail2ban-client status
sudo fail2ban-client status sshd

# Unban IP (ถ้าโดนเอง)
sudo fail2ban-client unban 192.168.1.50

# Restart
sudo systemctl restart fail2ban
```

### Layer 4: Automatic Security Updates

อย่าลืม update security patches — unattended-upgrades จะทำให้

```bash
# ติดตั้ง
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
# เลือก Yes → auto install security updates

# Config
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

```
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
};
Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Automatic-Reboot "false";  # ไม่ reboot auto
```

**Check status:**
```bash
sudo systemctl status unattended-upgrades
sudo cat /var/log/unattended-upgrades/unattended-upgrades.log
```

### Layer 5: Password Policy

บังคับความซับซ้อน password

```bash
# ติดตั้ง
sudo apt install libpam-pwquality

# Config
sudo nano /etc/security/pwquality.conf
```

```
minlen = 12                      # อย่างน้อย 12 ตัว
minclass = 3                     # ต้องมี 3 ประเภท (ตัวใหญ่/เล็ก/ตัวเลข)
maxrepeat = 3                    # ห้ามซ้ำ 3 ตัวติด
dcredit = -1                     # ต้องมีตัวเลข 1 ตัว
ucredit = -1                     # ต้องมีตัวใหญ่ 1 ตัว
lcredit = -1                     # ต้องมีตัวเล็ก 1 ตัว
```

### Layer 6: Additional Hardening

**Disable IPv6** (ถ้าไม่ใช้):
```bash
sudo nano /etc/sysctl.conf
```
```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
```

**Kernel Hardening:**
```bash
sudo nano /etc/sysctl.conf
```
```
# ลด swappiness
vm.swappiness = 10

# Network security
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
```

**Limit su access:**
```bash
# ให้เฉพาะ group wheel ใช้ su ได้
sudo dpkg-statoverride --update --add root root 4750 /bin/su
sudo chgrp wheel /bin/su
```

## Security Checklist

### Essential (ทำต้องทำ)
- [ ] SSH port เปลี่ยนแล้ว (หรือปิดเลยถ้าใช้ Tailscale)
- [ ] Password authentication ปิดแล้ว (SSH key only)
- [ ] Root login ปิดแล้ว
- [ ] UFW firewall เปิดแล้ว
- [ ] Fail2ban ทำงานอยู่
- [ ] unattended-upgrades เปิดแล้ว

### Recommended (แนะนำ)
- [ ] Password policy ตั้งแล้ว
- [ ] IPv6 ปิดถ้าไม่ใช้
- [ ] Kernel parameters hardening
- [ ] Different passwords สำหรับทุก service
- [ ] Two-factor auth (ถ้ารองรับ)
- [ ] Regular backups ทดสอบ restore แล้ว

### Advanced (ถ้าประสงค์)
- [ ] AppArmor / SELinux (MAC)
- [ ] Intrusion Detection System (IDS)
- [ ] File Integrity Monitoring (FIM)
- [ ] Security Information & Event Management (SIEM)
- [ ] Regular security audits

## Common Attacks & Prevention

| Attack | Description | Prevention |
|--------|-------------|-------------|
| **Brute Force** | ลอง password ทุก combo | Fail2ban + SSH key only |
| **DDoS** | flood requests | UFW rate limiting, Cloudflare |
| **Port Scanning** | scan open ports | Change ports, firewall |
| **Zero-day** | exploit ไม่รู้จัก | unattended-upgrades |
| **Insider Threat** | คนในบริษัท | Principle of least privilege |

## Monitoring & Detection

```bash
# ดู login attempts
sudo grep "Failed password" /var/log/auth.log

# ดู IP ที่ login เข้ามา
sudo last

# ดูใคร online ตอนนี้
sudo who

# ดู failed login
sudo lastb

# ดู ports ที่เปิด
ss -tlnp

# เช็คว่า IP โดน ban ไหม
sudo fail2ban-client status sshd
```

## SSL/TLS Certificates

ถ้า expose services ออก internet — **ต้องมี HTTPS**

```bash
# ติดตั้ง certbot
sudo apt install certbot python3-certbot-nginx

# ขอ certificate
sudo certbot --nginx -d yourdomain.com

# Renew อัตโนมัติ (ทุก 90 วัน)
sudo certbot renew --dry-run

# ดู certificates
sudo certbot certificates
```

## Gotchas

❌ **อย่าเปิด Port 22** ถ้าใช้ Tailscale — ใช้ Tailscale SSH แทน

❌ **อย่าใช้ password เดิม** ทุก service — ใช้ Vaultwarden (self-hosted Bitwarden)

❌ **อย่า expose database port** ออกนอก (postgres, mysql) — ให้อยู่ใน Docker network เท่านั้น

❌ **อย่าเก็บ API keys ใน code** หรือ git — ใช้ .env files

❌ **อย่าลืมทดสอบ backup** — restore ไม่ได้ = ไม่มีประโยชน์

## Tools Summary

| Tool | Purpose | Complexity |
|------|---------|------------|
| UFW | Firewall | ⭐ Simple |
| Fail2ban | IPS (ban brute force) | ⭐ Simple |
| unattended-upgrades | Auto patching | ⭐ Simple |
| Certbot | SSL certificates | ⭐ Simple |
| SSH keys | Authentication | ⭐ Simple |
| Tailscale | VPN + SSH | ⭐⭐ Medium |
| AppArmor | MAC (file access control) | ⭐⭐⭐ Advanced |
| SELinux | Advanced MAC | ⭐⭐⭐⭐ Expert |

## ระดับความปลอดภัยตาม Access Method

```
ระดับความปลอดภัย (1-10)

Port Forwarding แบบเปิดดิบ     [██░░░░░░░░] 2/10
Port Forward + Strong Password  [████░░░░░░] 4/10
Cloudflare Tunnel               [████████░░] 8/10
Tailscale VPN                   [█████████░] 9/10
Cloudflare + Tailscale ด้วยกัน  [██████████] 10/10
```

**Setup ที่แนะนำ (10/10):**
```
SSH (admin ใช้)     → Tailscale VPN ✅
Web Services        → Cloudflare Tunnel ✅
Database            → ไม่เปิดออกนอกเลย ✅
Docker network      → Isolated network ✅
UFW Firewall        → บล็อก port ที่ไม่ต้องการ ✅
```

## ความสัมพันธ์กับ concept อื่น

- **[[wiki/concepts/vpn-tailscale|VPN & Tailscale]]** — วิธีปลอดภัยแทน Port Forwarding สำหรับ SSH
- **[[wiki/concepts/cloudflare-tunnel|Cloudflare Tunnel]]** — วิธีปลอดภัยแทน Port Forwarding สำหรับ Web Services
- **[[wiki/concepts/port-networking|Port & Port Forwarding]]** — Port Forwarding เป็น access method ที่เสี่ยงที่สุด
- **[[wiki/concepts/ssl-tls|SSL/TLS & HTTPS]]** — HTTPS บังคับใช้เมื่อ expose services ออกอินเทอร์เน็ต

## แหล่งเรียนรู้เพิ่มเติม

- **Mozilla SSH Configuration** — mozilla.github.io/ssh-configuration
- **CIS Benchmarks** — security best practices มาตรฐานอุตสาหกรรม
- **SSH Audit** — sshaudit.com — scan SSH config หาช่องโหว่
- **haveibeenpwned.com** — เช็คว่า email โดน leak ไหม
- **shodan.io** — เช็คว่า IP บ้านเปิด port อะไรบ้าง (ก่อน attacker)
