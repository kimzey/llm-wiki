---
title: "SSL/TLS & HTTPS"
type: concept
tags: [ssl, tls, https, security, certificate, networking, cloudflare, home-server]
sources: [network-fundamentals.md]
related:
  - wiki/concepts/cloudflare-tunnel.md
  - wiki/concepts/reverse-proxy.md
  - wiki/concepts/server-security.md
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

SSL/TLS คือระบบเข้ารหัส traffic ระหว่างผู้ใช้กับ server — HTTPS = HTTP + TLS ทำให้ข้อมูลที่ส่งผ่านอินเทอร์เน็ต (password, personal data) อ่านไม่ออกถ้าถูกดัก SSL Certificate คือ "ใบรับรอง" ว่าเว็บนั้นเป็นของจริง

## อธิบาย

### HTTP vs HTTPS

```
HTTP  (ไม่มี SSL) = ส่งข้อมูลแบบ plain text
─────────────────────────────────────────────
คุณ ──[username:admin password:1234]──→ Server
         ↑
    ใครดัก traffic ได้เห็นหมดเลย ⚠️

HTTPS (มี SSL) = ข้อมูลเข้ารหัสก่อนส่ง
─────────────────────────────────────────────
คุณ ──[xK92#$mP!q8@...]──→ Server
         ↑
    ดักได้แต่อ่านไม่ออก ✅
```

### SSL Certificate คืออะไร

ใบรับรองที่บอกว่า:
1. "เว็บนี้เป็นของจริง ไม่ใช่ของปลอม (man-in-the-middle)"
2. Browser เชื่อมต่อมาถูก server จริงๆ
3. ออกโดย Certificate Authority (CA) ที่เชื่อถือได้

**CA ฟรีสำหรับ Home Server:**
- **Let's Encrypt** — CA ฟรีที่ได้รับการยอมรับทั่วโลก, ต้อง renew ทุก 90 วัน (auto ได้)
- **Cloudflare** — ออกให้อัตโนมัติเมื่อใช้ Cloudflare Tunnel หรือ Cloudflare proxy

## ประเด็นสำคัญ

### TLS Handshake (ย่อ)

1. Client ส่ง hello พร้อม algorithms ที่รองรับ
2. Server ส่ง Certificate กลับ
3. Client ตรวจสอบ Certificate กับ CA
4. แลก key → สร้าง session key
5. ส่งข้อมูลเข้ารหัสด้วย session key

### SSL ใน Home Server Setup

**วิธีที่ 1: Cloudflare (แนะนำ)**
- ใช้ Cloudflare Tunnel หรือ Cloudflare proxy
- Cloudflare ออก Certificate อัตโนมัติ ฟรี
- Renew อัตโนมัติ ไม่ต้องสนใจ

**วิธีที่ 2: Let's Encrypt + Certbot**
- ใช้กับ Nginx หรือ Caddy
- สั่ง `certbot --nginx -d yourdomain.com`
- Certbot renew อัตโนมัติผ่าน cron job

**วิธีที่ 3: Self-signed Certificate**
- สร้างเอง ไม่ต้องพึ่ง CA
- Browser จะแสดง warning (ไม่ trusted)
- ใช้ได้ใน LAN หรือ Tailscale เท่านั้น

### ทำไม HTTPS จำเป็น แม้ใน Home Server

- Password ที่ type ใน Home Assistant, Portainer จะถูกเข้ารหัส
- ป้องกัน MITM attack บน LAN เอง
- Modern browser บล็อก mixed content (HTTP + HTTPS บนหน้าเดียว)
- Service workers, Push notifications ต้องการ HTTPS

## ตัวอย่าง / กรณีศึกษา

Home Server ใช้ Cloudflare Tunnel → SSL จัดการอัตโนมัติ:
- `ha.yourdomain.com` → HTTPS ✅ (Cloudflare Certificate)
- ผู้ใช้เห็น 🔒 padlock ใน browser
- ไม่ต้อง config certbot หรือ renew เอง

## ความสัมพันธ์กับ concept อื่น

- **[[wiki/concepts/cloudflare-tunnel|Cloudflare Tunnel]]** — ออก SSL Certificate อัตโนมัติ
- **[[wiki/concepts/reverse-proxy|Reverse Proxy]]** — Nginx terminate SSL และส่งต่อ traffic ภายใน
- **[[wiki/concepts/server-security|Server Security]]** — HTTPS เป็นส่วนหนึ่งของ security layers

## แหล่งที่มา

- [[wiki/sources/network-fundamentals|พื้นฐาน Network & Infrastructure]] — section 10
