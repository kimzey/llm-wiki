---
title: "VPN & Tailscale"
type: concept
tags: [vpn, tailscale, wireguard, networking, home-server, security, infrastructure]
sources: [network-fundamentals.md]
related:
  - wiki/concepts/ip-address-networking.md
  - wiki/concepts/port-networking.md
  - wiki/concepts/cloudflare-tunnel.md
  - wiki/concepts/server-security.md
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

VPN (Virtual Private Network) สร้าง "อุโมงค์เข้ารหัส" ระหว่าง device — Tailscale คือ VPN แบบ Mesh ที่ใช้ WireGuard protocol ทำให้เชื่อมต่อ device ต่างๆ ได้เหมือนอยู่ใน LAN เดียวกัน โดยไม่ต้อง port forward และไม่ต้อง config router

## อธิบาย

### Tailscale Architecture

```
NUC (ในบ้าน)                    Laptop (นอกบ้าน)
192.168.1.100                   (IP อะไรก็ได้)
100.64.0.5 ←── Tailscale ───→ 100.64.0.2
               Mesh VPN
               (WireGuard encrypted)

Tailscale Coordination Server
(แค่ช่วย handshake ครั้งแรก — หลังจากนั้น
 device คุยกันตรงๆ ไม่ผ่าน server)
```

Tailscale ใช้ IP range `100.64.x.x` สำหรับ device ทุกตัวใน network — เรียกว่า "Tailnet"

### การใช้งาน

หลัง connect Tailscale แล้ว:
```bash
ssh admin@100.64.0.5   # เหมือนอยู่ในบ้านเลย
```

## ประเด็นสำคัญ

### ทำไม Tailscale ปลอดภัย

- ทุก connection เข้ารหัสด้วย **WireGuard** (military-grade encryption)
- ต้อง login ด้วย Google/GitHub account ก่อนใช้
- คนนอกที่ไม่ได้ login account เดียวกัน เข้าไม่ได้เลย
- ไม่ต้องเปิด Port ใดๆ บน Router — NUC เป็น **outbound connection** ออกไปหา Tailscale

### WireGuard Protocol

WireGuard คือ VPN protocol รุ่นใหม่ที่:
- เร็วกว่า OpenVPN และ IPSec มาก
- Code base เล็กมาก (~4,000 บรรทัด) → audit ง่าย, ช่องโหว่น้อย
- Built into Linux kernel ตั้งแต่ kernel 5.6
- Tailscale ใช้ WireGuard เป็น transport layer

### เปรียบเทียบ VPN vs Cloudflare Tunnel

| ด้าน | VPN/Tailscale | Cloudflare Tunnel |
|------|--------------|------------------|
| Use case | SSH, dev work ส่วนตัว | Web services, แชร์ผู้ใช้ |
| ผู้เข้าถึง | เฉพาะ device ในบ้าน | ทุกคนผ่าน domain (ต้อง login) |
| Port ที่เปิด | ไม่มีเลย | ไม่มีเลย |
| IP ซ่อน | ใช่ | ใช่ |
| Third-party | Tailscale server (handshake) | Cloudflare edge |

### ข้อจำกัด

- ต้องติดตั้ง Tailscale ทุก device ที่ต้องการเข้าถึง
- ถ้า ISP บล็อก UDP (บางที่) อาจช้ากว่าปกติ (Tailscale จะ fallback ผ่าน DERP relay)

## ตัวอย่าง / กรณีศึกษา

Home Server Plan:
```
SSH (admin ใช้)     → Tailscale VPN ✅ ปลอดภัยมาก
Web Services        → Cloudflare Tunnel ✅ ปลอดภัยมาก
Database            → ไม่เปิดออกนอกเลย ✅ ปลอดภัยสุด
```

ระดับความปลอดภัย: **Tailscale = 9/10**

## ความสัมพันธ์กับ concept อื่น

- **[[wiki/concepts/ip-address-networking|IP Address & Networking]]** — Tailscale ใช้ IP range `100.64.x.x` (Carrier-grade NAT range)
- **[[wiki/concepts/port-networking|Port & Port Forwarding]]** — Tailscale แทนที่ Port Forwarding อย่างปลอดภัย
- **[[wiki/concepts/cloudflare-tunnel|Cloudflare Tunnel]]** — ใช้คู่กัน: Tailscale สำหรับ admin, Cloudflare สำหรับ public web
- **[[wiki/concepts/server-security|Server Security]]** — ส่วนหนึ่งของ security stack: Tailscale + Cloudflare + UFW

## แหล่งที่มา

- [[wiki/sources/network-fundamentals|พื้นฐาน Network & Infrastructure]] — sections 7, 8, 14
