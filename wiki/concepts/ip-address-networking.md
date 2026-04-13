---
title: "IP Address & Networking"
type: concept
tags: [networking, ip-address, nat, public-ip, private-ip, home-server, infrastructure]
sources: [network-fundamentals.md]
related:
  - wiki/concepts/port-networking.md
  - wiki/concepts/dns.md
  - wiki/concepts/vpn-tailscale.md
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

IP Address คือ "ที่อยู่" ของแต่ละ device บนเครือข่าย มี 2 ประเภท: **Public IP** (ที่อยู่บ้านทั้งหลังบนอินเทอร์เน็ต) และ **Private IP** (ที่อยู่ห้องแต่ละห้องภายในบ้าน) — Router ทำหน้าที่แปลงระหว่าง 2 ประเภทนี้ด้วยกระบวนการที่เรียกว่า NAT

## อธิบาย

### โครงสร้างเครือข่ายบ้าน

```
อินเทอร์เน็ต (โลกภายนอก)
        │
        │  ← สาย Fiber / ADSL จาก ISP (AIS/TRUE/DTAC)
        │
   [ Router/Modem ]  ← เครื่องกล่องที่ได้จาก ISP
        │
   ─────┼─────────────────────
        │         │         │
      NUC      มือถือ    Laptop
  192.168.1.100  .101      .102
```

Router เป็น "ประตูบ้าน" — แปลง Public IP 1 อันให้กระจายเป็น Private IP หลายอันให้ทุก device ในบ้าน

### IP Address 2 ประเภท

**Public IP (IP สาธารณะ)**
- ที่อยู่ "บ้านทั้งหลัง" บนอินเทอร์เน็ต
- ทุกคนในโลกเห็นได้
- ISP กำหนดให้ เปลี่ยนได้ตลอด (Dynamic IP)
- ตัวอย่าง: `101.51.147.23`

**Private IP (IP ส่วนตัว)**
- ที่อยู่ "ห้องในบ้าน" แต่ละ device
- คนนอกบ้านมองไม่เห็น
- Router กำหนดให้
- ตัวอย่าง: `192.168.1.100` (NUC), `192.168.1.101` (มือถือ)

### ช่วง IP ที่เป็น Private เสมอ

```
192.168.0.0   →  192.168.255.255   (บ้านทั่วไป)
10.0.0.0      →  10.255.255.255    (องค์กร)
172.16.0.0    →  172.31.255.255    (Docker ใช้)
100.64.0.0    →  100.127.255.255   (Tailscale Mesh)
```

## ประเด็นสำคัญ

### NAT — Network Address Translation

NAT คือกระบวนการที่ Router ใช้แปลง IP เวลาส่งข้อมูลออกนอกและรับกลับ:

```
NUC (192.168.1.100) ส่ง request ออกไป Google
           ↓
Router รับ → เปลี่ยน IP เป็น Public IP บ้าน (101.51.147.23)
           ↓
Google เห็นแค่ IP บ้าน ไม่รู้ว่า device ไหน
           ↓
Google ตอบกลับมาที่ 101.51.147.23
           ↓
Router รู้ว่าต้องส่งต่อให้ 192.168.1.100
```

**ผลสำคัญของ NAT:** ถ้าคนนอกส่งมาหา Public IP โดยตรง Router ไม่รู้จะส่งต่อให้ device ไหน → ทิ้งทันที → **คนนอกบ้านเข้า Server ไม่ได้โดยตรง**

### Dynamic vs Static Public IP

- ISP ส่วนใหญ่ให้ Dynamic IP — เปลี่ยนได้ทุกเมื่อ
- ไม่มีผลกับ Cloudflare Tunnel และ Tailscale เพราะไม่ได้ใช้ Public IP โดยตรง

## ตัวอย่าง / กรณีศึกษา

Home Server NUC7i5BNH ใช้ IP:
- Private IP: `192.168.1.100` (ภายในบ้าน)
- Tailscale IP: `100.64.0.5` (ภายใน Tailscale network)
- Public IP: Dynamic (ไม่ใช้โดยตรง — ใช้ผ่าน Cloudflare Tunnel แทน)

## ความสัมพันธ์กับ concept อื่น

- **[[wiki/concepts/port-networking|Port & Port Forwarding]]** — Port ทำงานร่วมกับ IP ในการระบุ service ปลายทาง
- **[[wiki/concepts/dns|DNS]]** — แปลง domain name เป็น IP Address
- **[[wiki/concepts/vpn-tailscale|VPN & Tailscale]]** — สร้าง Virtual Private Network ด้วย IP range `100.64.x.x`
- **[[wiki/concepts/cloudflare-tunnel|Cloudflare Tunnel]]** — วิธีซ่อน Public IP ด้วย reverse tunnel

## แหล่งที่มา

- [[wiki/sources/network-fundamentals|พื้นฐาน Network & Infrastructure]] — sections 1, 2, 5
