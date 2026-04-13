---
title: "Reverse Proxy"
type: concept
tags: [reverse-proxy, nginx, networking, home-server, infrastructure, https]
sources: [wiki/sources/network-fundamentals, wiki/sources/homeserver-admin-knowledge]
related:
  - wiki/concepts/ssl-tls.md
  - wiki/concepts/cloudflare-tunnel.md
  - wiki/concepts/port-networking.md
  - wiki/concepts/docker-containers.md
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

Reverse Proxy คือ "พนักงานต้อนรับ" ที่รับ request ทั้งหมดที่ port 443 (HTTPS) แล้วส่งต่อไปยัง service ที่ถูกต้องตาม domain name — แก้ปัญหาที่มี services หลายตัวแต่ port 443 มีแค่อันเดียว

## อธิบาย

### ปัญหาที่ Reverse Proxy แก้

```
ถ้าไม่มี Reverse Proxy:
ha.yourdomain.com:8123    ← ผิด! HTTPS ต้องใช้ port 443
portainer.yourdomain.com:9000  ← ผิด!

ถ้ามี Reverse Proxy (Nginx):
ha.yourdomain.com:443  → Nginx → localhost:8123
portainer.yourdomain.com:443 → Nginx → localhost:9000
n8n.yourdomain.com:443 → Nginx → localhost:5678
api.yourdomain.com:443 → Nginx → localhost:3000
```

Nginx ดู **domain name** ใน HTTP header เพื่อตัดสินใจว่าจะส่งต่อไปไหน

### Flow การทำงาน

```
ผู้ใช้ → ha.yourdomain.com (port 443)
              ↓
         Nginx (reverse proxy)
         ตรวจ Host header: "ha.yourdomain.com"
              ↓
         ส่งต่อไป localhost:8123
              ↓
         Home Assistant ตอบกลับ
              ↓
         Nginx ส่งกลับผู้ใช้
```

## ประเด็นสำคัญ

### Nginx เป็น Reverse Proxy ที่นิยมใช้

```nginx
# ตัวอย่าง Nginx config
server {
    listen 443 ssl;
    server_name ha.yourdomain.com;

    location / {
        proxy_pass http://localhost:8123;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### ทางเลือก Reverse Proxy อื่น

| Tool | เหมาะกับ |
|------|----------|
| **Nginx** | standard, config-based, performant |
| **Nginx Proxy Manager** | UI-friendly, เหมาะ Home Server |
| **Caddy** | Auto HTTPS, config ง่าย |
| **Traefik** | Docker-native, auto-detect containers |

### Reverse Proxy vs Cloudflare Tunnel

Cloudflare Tunnel ทำหน้าที่ reverse proxy ด้วยในระดับหนึ่ง — แต่:
- **Cloudflare Tunnel**: จัดการ external → NUC (internet-facing)
- **Nginx**: จัดการ internal routing บน NUC (service → service)

ในหลาย setup ใช้ทั้งคู่: Cloudflare → Nginx → Services

### Nginx Proxy Manager (NPM)

สำหรับ Home Server แนะนำ Nginx Proxy Manager — ทำงานผ่าน Web UI:
- เพิ่ม host ใหม่ด้วย UI ไม่ต้อง edit config text
- Manage SSL Certificates ผ่าน UI
- รันเป็น Docker container

## ตัวอย่าง / กรณีศึกษา

Home Server ใช้ Cloudflare Tunnel → Nginx Proxy Manager → Docker Services:
```
อินเทอร์เน็ต
    ↓ HTTPS
Cloudflare Tunnel
    ↓
Nginx Proxy Manager (port 443)
    ├── ha.domain.com → homeassistant:8123
    ├── portainer.domain.com → portainer:9000
    └── n8n.domain.com → n8n:5678
```

## ความสัมพันธ์กับ concept อื่น

- **[[wiki/concepts/ssl-tls|SSL/TLS & HTTPS]]** — Nginx terminate SSL และส่ง plain HTTP ภายใน
- **[[wiki/concepts/cloudflare-tunnel|Cloudflare Tunnel]]** — Cloudflare → Nginx เป็น common pattern
- **[[wiki/concepts/port-networking|Port & Port Forwarding]]** — Reverse Proxy ทำให้ไม่ต้อง port forward แต่ละ service
- **[[wiki/concepts/docker-containers|Docker Containers]]** — Nginx Proxy Manager รันใน Docker

## แหล่งที่มา

- [[wiki/sources/network-fundamentals|พื้นฐาน Network & Infrastructure]] — section 11
