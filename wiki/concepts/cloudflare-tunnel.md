---
title: "Cloudflare Tunnel"
type: concept
tags: [cloudflare, tunnel, reverse-tunnel, networking, home-server, security, infrastructure]
sources: [wiki/sources/network-fundamentals, wiki/sources/homeserver-admin-knowledge]
related:
  - wiki/concepts/ip-address-networking.md
  - wiki/concepts/port-networking.md
  - wiki/concepts/dns.md
  - wiki/concepts/vpn-tailscale.md
  - wiki/concepts/ssl-tls.md
  - wiki/concepts/reverse-proxy.md
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

Cloudflare Tunnel (เดิมชื่อ Argo Tunnel) คือ Reverse Tunnel ที่ให้ NUC เปิด outbound connection ออกไปหา Cloudflare Edge — ทำให้ผู้ใช้เข้าถึง services ผ่าน domain ได้โดยไม่ต้อง port forward และ IP บ้านไม่เปิดเผยเลย

## อธิบาย

### กลไกการทำงาน

```
ขั้นตอนที่ 1: NUC เปิด "อุโมงค์" ออกไปหา Cloudflare
─────────────────────────────────────────────────
NUC ──── outbound connection ────→ Cloudflare Edge
(NUC เป็นคนเปิด ไม่ใช่คนนอก → ไม่ต้องเปิด port!)

ขั้นตอนที่ 2: ผู้ใช้เข้า domain
─────────────────────────────────────────────────
ผู้ใช้ → ha.yourdomain.com
              ↓
         Cloudflare รับ (HTTPS ✅)
              ↓
         ส่งผ่านอุโมงค์ที่ NUC เปิดไว้
              ↓
         Home Assistant ตอบกลับ
              ↑
         ย้อนกลับผ่าน tunnel เดิม
              ↓
         ผู้ใช้ได้รับข้อมูล
```

**IP บ้านคุณไม่โผล่ที่ไหนเลย** ✅

### ข้อดีเปรียบ Port Forwarding

| ด้าน | Port Forwarding | Cloudflare Tunnel |
|------|----------------|------------------|
| Port ที่เปิดบน Router | ต้องเปิด | ไม่มีเลย |
| IP บ้านเปิดเผย | ใช่ | ไม่ |
| SSL Certificate | จัดการเอง | อัตโนมัติ ฟรี |
| Dynamic IP ปัญหา | ใช่ | ไม่มีผล |
| Bot สแกน | โดน | ไม่โดน |
| ความปลอดภัย | 2/10 | 8/10 |

## ประเด็นสำคัญ

### ทำไมปลอดภัยกว่า

- NUC เป็น initiator — เปิดการเชื่อมต่อออกไปเอง ไม่รับ inbound
- คนนอกไม่รู้ IP บ้าน ไม่รู้ว่ามี server อยู่ที่บ้าน
- Cloudflare เป็น reverse proxy รับ DDoS แทน
- SSL/TLS terminated ที่ Cloudflare Edge

### การตั้งค่า (cloudflared)

`cloudflared` คือ daemon ที่รันบน NUC และรักษา tunnel ให้เปิดตลอด:

```bash
# ติดตั้ง cloudflared และ login
cloudflared tunnel login

# สร้าง tunnel
cloudflared tunnel create homelab

# config routing
# yourdomain.com → localhost:8123 (Home Assistant)
# portainer.yourdomain.com → localhost:9000
```

### คำถามที่มักสงสัย

**"ถ้าใช้ Cloudflare Tunnel ใครๆ ก็เข้าได้ไหม?"**
```
เข้าถึง URL ได้ ≠ ทำอะไรได้

ha.yourdomain.com เข้าได้ทุกคน
แต่ต้อง Login ด้วย username + password ก่อน
ถ้าไม่มี account → ไม่เห็นอะไรเลย ✅
```

**"ถ้าไฟดับหรือ NUC ปิด Tunnel จะขาดไหม?"**
ขาดครับ แต่พอเปิดใหม่ก็กลับมาทันที (แนะนำใช้ UPS)

**"ISP เปลี่ยน Public IP มีผลไหม?"**
ไม่มีผลเลย — ไม่ได้ใช้ Public IP โดยตรง ✅

### ข้อเสีย

- Traffic ผ่าน Cloudflare (third-party) — Cloudflare เห็น unencrypted traffic ถ้าไม่ใช้ end-to-end encryption
- ต้องพึ่งพา Cloudflare service (free tier มี rate limit)

## ตัวอย่าง / กรณีศึกษา

Home Server Plan ใช้ Cloudflare Tunnel สำหรับ:
- `ha.yourdomain.com` → Home Assistant `:8123`
- `portainer.yourdomain.com` → Portainer `:9000`
- `n8n.yourdomain.com` → n8n `:5678`

Tailscale ใช้ต่างหากสำหรับ SSH และ dev work ส่วนตัว

**ระดับความปลอดภัยรวม: Cloudflare + Tailscale = 10/10**

## ความสัมพันธ์กับ concept อื่น

- **[[wiki/concepts/dns|DNS]]** — Domain ชี้ไป Cloudflare แทน IP บ้าน
- **[[wiki/concepts/ssl-tls|SSL/TLS & HTTPS]]** — Cloudflare ออก SSL Certificate อัตโนมัติ
- **[[wiki/concepts/reverse-proxy|Reverse Proxy]]** — Cloudflare ทำหน้าที่ reverse proxy ก่อนถึง NUC
- **[[wiki/concepts/vpn-tailscale|VPN & Tailscale]]** — ใช้คู่กัน: Cloudflare สำหรับ public web, Tailscale สำหรับ admin
- **[[wiki/concepts/port-networking|Port & Port Forwarding]]** — Cloudflare Tunnel แทนที่ Port Forwarding

## แหล่งที่มา

- [[wiki/sources/infrastructure/network-fundamentals|พื้นฐาน Network & Infrastructure]] — sections 7, 9, 14, 15
