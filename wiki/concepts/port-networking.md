---
title: "Port & Port Forwarding"
type: concept
tags: [networking, port, port-forwarding, home-server, security, infrastructure]
sources: [wiki/sources/network-fundamentals, wiki/sources/homeserver-admin-knowledge]
related:
  - wiki/concepts/ip-address-networking.md
  - wiki/concepts/server-security.md
  - wiki/concepts/cloudflare-tunnel.md
  - wiki/concepts/vpn-tailscale.md
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

Port คือ "ประตูห้อง" ภายในบ้าน (device) แต่ละ Port ตรงกับโปรแกรมหนึ่งตัว — ถ้า IP Address = บ้าน, Port = ประตูห้องต่างๆ ในบ้าน Port Forwarding คือการบอก Router ว่าให้ส่ง traffic จากนอกบ้านเข้ามาหา device ภายในผ่าน port ที่กำหนด

## อธิบาย

### Port คืออะไร

```
บ้านเลขที่ 192.168.1.100 (NUC)
├── ประตู 22    → SSH (เข้า Terminal)
├── ประตู 80    → HTTP (เว็บปกติ)
├── ประตู 443   → HTTPS (เว็บมี SSL)
├── ประตู 8123  → Home Assistant
├── ประตู 9000  → Portainer
├── ประตู 3001  → Uptime Kuma
└── ประตู 5678  → n8n
```

- **Port ที่ปิดอยู่** = ไม่มีใครเข้าได้แม้รู้ IP
- **Port ที่เปิดอยู่** = มีโปรแกรมรอรับการเชื่อมต่อ (Listening)

### Well-known Ports (Port มาตรฐาน)

| Port | Protocol | ใช้ทำอะไร |
|------|----------|-----------|
| 21   | FTP      | File Transfer |
| 22   | SSH      | Secure Shell (remote terminal) |
| 25   | SMTP     | Email ขาออก |
| 53   | DNS      | Domain Name Resolution |
| 80   | HTTP     | เว็บปกติ |
| 443  | HTTPS    | เว็บมี SSL |
| 3306 | MySQL    | Database |
| 5432 | PostgreSQL | Database |

## ประเด็นสำคัญ

### Port Forwarding คืออะไร

บอก Router ว่า "ถ้ามีคนมาที่ port นี้ ส่งต่อให้ device นี้เลย"

```
คนนอกบ้าน → 101.51.147.23:8123
                   ↓
Router เห็น port 8123 → ส่งต่อให้ 192.168.1.100:8123
                   ↓
Home Assistant ตอบกลับ ✅
```

### ⚠️ ความเสี่ยงของ Port Forwarding

**อันตรายมาก** — Bot ทั่วโลกสแกนหา Port ที่เปิดอยู่ตลอดเวลา:

```
อินเทอร์เน็ต
     │
     ▼
101.51.147.23:22  ← Bot สแกนทั่วโลกตลอดเวลา!
     │               พยายาม brute force password
     ▼
NUC ของคุณ  ← ถ้า password อ่อนหรือมีช่องโหว่ = โดนแฮก
```

**Port ที่อันตรายถ้าเปิดออกอินเทอร์เน็ต:**
- Port 22 (SSH) — โดนสแกนและ brute force ตลอด
- Port 3389 (RDP) — อันตรายมาก
- Port 3306/5432 (Database) — ห้ามเปิดเด็ดขาด

### เปรียบเทียบ 3 วิธีเข้า Server จากนอกบ้าน

| วิธี | ความปลอดภัย | ความซับซ้อน | เหมาะกับ |
|------|------------|-------------|----------|
| Port Forwarding | 2/10 | ต่ำ | legacy, ไม่มีทางเลือก |
| VPN/Tailscale | 9/10 | กลาง | SSH, dev work ส่วนตัว |
| Cloudflare Tunnel | 8/10 | กลาง | Web services, แชร์ผู้ใช้ |

## ตัวอย่าง / กรณีศึกษา

Home Server Setup (ปลอดภัย):
- SSH → **Tailscale** (ไม่ port forward เลย)
- Web Services → **Cloudflare Tunnel** (ไม่ port forward เลย)
- Database → ไม่เปิดออกนอกเลย
- ผลลัพธ์: Router ไม่ต้องเปิด Port ใดๆ ✅

## ความสัมพันธ์กับ concept อื่น

- **[[wiki/concepts/ip-address-networking|IP Address & Networking]]** — IP + Port ระบุ service ปลายทางร่วมกัน
- **[[wiki/concepts/server-security|Server Security]]** — UFW Firewall บล็อก Port ที่ไม่ต้องการ
- **[[wiki/concepts/vpn-tailscale|VPN & Tailscale]]** — ทางเลือกที่ปลอดภัยกว่า Port Forwarding
- **[[wiki/concepts/cloudflare-tunnel|Cloudflare Tunnel]]** — ทางเลือกที่ปลอดภัยสำหรับ Web Services

## แหล่งที่มา

- [[wiki/sources/network-fundamentals|พื้นฐาน Network & Infrastructure]] — sections 3, 6, 7
