---
title: "DNS — Domain Name System"
type: concept
tags: [networking, dns, domain, home-server, infrastructure, cloudflare]
sources: [wiki/sources/network-fundamentals, wiki/sources/homeserver-admin-knowledge]
related:
  - wiki/concepts/ip-address-networking.md
  - wiki/concepts/cloudflare-tunnel.md
  - wiki/concepts/ssl-tls.md
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

DNS (Domain Name System) คือ "สมุดโทรศัพท์" ของอินเทอร์เน็ต — แปลง domain name ที่มนุษย์จำได้ (เช่น `google.com`) ให้เป็น IP Address ที่คอมพิวเตอร์ใช้คุยกัน (เช่น `142.250.185.46`)

## อธิบาย

คอมพิวเตอร์คุยกันด้วย IP Address แต่คนจำ IP ไม่ได้ DNS จึงทำหน้าที่แปลง:

```
คุณพิมพ์: google.com
     ↓
DNS Server ค้นหา
     ↓
ได้ IP: 142.250.185.46
     ↓
Browser เชื่อมต่อไปที่ IP นั้น
```

### DNS ทำงานอย่างไร

1. Browser ถาม DNS Resolver ว่า `google.com` คือ IP อะไร
2. DNS Resolver ค้นหาใน cache ก่อน ถ้าไม่มี → ถาม Root DNS Server
3. ได้ IP กลับมา → Browser เชื่อมต่อไป IP นั้น
4. DNS cache ผลไว้ชั่วคราว (TTL) เพื่อไม่ต้องถามซ้ำ

## ประเด็นสำคัญ

### DNS ของ Home Server

```
ha.yourdomain.com  →  (ชี้ไปที่ Cloudflare Tunnel)
                          ↓
                    Cloudflare รู้ว่าต้องส่งต่อไปที่ NUC
                          ↓
                    192.168.1.100:8123
```

Domain ชี้ไปที่ Cloudflare (ไม่ใช่ Public IP บ้านโดยตรง) → IP บ้านไม่เปิดเผย

### DNS Record Types ที่สำคัญ

| Record | ความหมาย |
|--------|----------|
| A     | domain → IPv4 address |
| AAAA  | domain → IPv6 address |
| CNAME | domain → อีก domain หนึ่ง (alias) |
| MX    | domain → mail server |
| TXT   | ข้อมูล text (verification, SPF) |

### DNS Provider ที่ใช้กับ Home Server

- **Cloudflare DNS**: จัดการ DNS + ออก SSL Certificate อัตโนมัติ + Tunnel integration
- Domain ชี้ไป Cloudflare → Cloudflare route ผ่าน Tunnel ไปยัง NUC

## ตัวอย่าง / กรณีศึกษา

เมื่อ ISP เปลี่ยน Public IP บ้าน (Dynamic IP):
- **Port Forwarding**: กระทบทันที ต้องอัปเดต DNS ใหม่
- **Cloudflare Tunnel**: ไม่กระทบเลย — NUC เปิด outbound connection ออกไปหา Cloudflare ไม่ได้ใช้ Public IP
- **Tailscale VPN**: ไม่กระทบเลย — ใช้ Tailscale IP range `100.64.x.x` แทน

## ความสัมพันธ์กับ concept อื่น

- **[[wiki/concepts/ip-address-networking|IP Address & Networking]]** — DNS แปลง domain เป็น IP
- **[[wiki/concepts/cloudflare-tunnel|Cloudflare Tunnel]]** — DNS ชี้ไป Cloudflare แทน IP บ้าน
- **[[wiki/concepts/ssl-tls|SSL/TLS & HTTPS]]** — Cloudflare ออก SSL Certificate ผ่าน DNS

## แหล่งที่มา

- [[wiki/sources/network-fundamentals|พื้นฐาน Network & Infrastructure]] — section 4
