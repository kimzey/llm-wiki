---
title: "พื้นฐาน Network & Infrastructure"
type: source
source_file: raw/notes/network-fundamentals.md
url: ""
published: 2026-04-13
tags: [networking, ip-address, dns, nat, vpn, cloudflare, tailscale, home-server, infrastructure, firewall, ssl, reverse-proxy]
related:
  - wiki/concepts/ip-address-networking.md
  - wiki/concepts/port-networking.md
  - wiki/concepts/dns.md
  - wiki/concepts/vpn-tailscale.md
  - wiki/concepts/cloudflare-tunnel.md
  - wiki/concepts/ssl-tls.md
  - wiki/concepts/reverse-proxy.md
  - wiki/concepts/docker-containers.md
  - wiki/concepts/server-security.md
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/network-fundamentals.md|Original file]]

## สรุป

คู่มืออธิบายพื้นฐาน Network & Infrastructure สำหรับ Home Server ตั้งแต่ศูนย์ — เหมาะสำหรับคนที่ไม่รู้อะไรเลย ครอบคลุม IP Address, Port, DNS, NAT, Port Forwarding, VPN (Tailscale), Cloudflare Tunnel, SSL/HTTPS, Reverse Proxy, Docker Network, และ Firewall (UFW)

## ประเด็นสำคัญ

- **IP Address 2 ประเภท**: Public IP (ที่อยู่บ้านทั้งหลังบนอินเทอร์เน็ต) และ Private IP (192.168.x.x ภายในบ้าน — คนนอกมองไม่เห็น)
- **Port = ประตูห้อง**: IP คือบ้าน, Port คือประตูห้องแต่ละห้อง — เช่น port 22 (SSH), 80 (HTTP), 443 (HTTPS), 8123 (Home Assistant)
- **NAT ป้องกันการเข้าจากนอก**: Router แปลง IP ทำให้คนนอกไม่รู้ว่า device ไหนอยู่ในบ้าน — ต้องใช้วิธีพิเศษถึงจะเข้า server จากนอกบ้านได้
- **3 วิธีเข้า server จากนอกบ้าน**: (1) Port Forwarding (เก่า/เสี่ยง), (2) VPN/Tailscale (ปลอดภัย), (3) Cloudflare Tunnel (ปลอดภัยที่สุด/ทันสมัย)
- **Tailscale** ใช้ WireGuard encryption — ต้อง login account เดียวกันถึงเข้าได้, ไม่ expose port ใดๆ
- **Cloudflare Tunnel**: NUC เปิด outbound connection ออกไปหา Cloudflare — ไม่ต้อง port forward, IP บ้านซ่อนทั้งหมด
- **Reverse Proxy (Nginx)**: รับ request ที่ port 443 แล้วส่งต่อไป service ต่างๆ ตาม domain name
- **Docker Network**: Container ใน network เดียวกันคุยกันได้ผ่านชื่อ, ต่าง network คุยกันไม่ได้
- **ระดับความปลอดภัย**: Cloudflare Tunnel + Tailscale = 10/10, Port Forward ธรรมดา = 2/10

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- Private IP ranges: `192.168.x.x` (บ้าน), `10.x.x.x` (องค์กร), `172.16-31.x.x` (Docker ใช้)
- Tailscale ใช้ IP range `100.64.x.x` สำหรับ mesh network
- Port 22 (SSH) โดน bot สแกนและ brute force ตลอดเวลาจากทั่วโลก — ห้าม Port Forward
- Well-known ports: 21 FTP, 22 SSH, 25 SMTP, 53 DNS, 80 HTTP, 443 HTTPS, 3306 MySQL, 5432 PostgreSQL

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่มีข้อขัดแย้ง — เป็นความรู้พื้นฐาน network ที่เป็นมาตรฐาน

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/ip-address-networking|IP Address & Networking]] — Public/Private IP, NAT, Private ranges
- [[wiki/concepts/port-networking|Port & Port Forwarding]] — Port concepts, well-known ports, Port Forwarding risks
- [[wiki/concepts/dns|DNS]] — Domain Name System, การแปลง domain เป็น IP
- [[wiki/concepts/vpn-tailscale|VPN & Tailscale]] — WireGuard VPN Mesh, Tailscale architecture
- [[wiki/concepts/cloudflare-tunnel|Cloudflare Tunnel]] — Reverse tunnel, outbound connection, ซ่อน IP
- [[wiki/concepts/ssl-tls|SSL/TLS & HTTPS]] — Certificate, encryption, Let's Encrypt
- [[wiki/concepts/reverse-proxy|Reverse Proxy]] — Nginx, domain-based routing, single port 443
- [[wiki/concepts/docker-containers|Docker Containers]] — Docker Network isolation
- [[wiki/concepts/server-security|Server Security]] — UFW Firewall, port blocking
