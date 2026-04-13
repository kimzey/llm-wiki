---
title: "Docker Containers"
type: concept
tags: [docker, containers, orchestration, devops, networking]
sources: [wiki/sources/homeserver-admin-knowledge, wiki/sources/network-fundamentals]
related:
  - wiki/concepts/reverse-proxy.md
  - wiki/concepts/ip-address-networking.md
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

Docker คือเทคโนโลยี containerization ที่ให้บรรจุ application พร้อม environment ทั้งหมดไว้ใน "container" หนึ่ง — ทำให้ deploy ง่าย, isolate กัน, และ consistent ทุก environment

## อธิบาย

Docker Container คือ lightweight, standalone, executable package ที่บรรจุทุกอย่างที่ app ต้องการ: code, runtime, system tools, libraries, settings ต่างๆ — แตกต่างจาก VM ที่ต้องการ OS ทั้งตัว

### ส่วนประกอบหลัก
- **Image**: Template สำหรับสร้าง container (read-only)
- **Container**: Instance ที่รันอยู่จาก image
- **Dockerfile**: Script สร้าง image
- **Docker Compose**: Tool จัดการ multi-container ด้วย YAML file

## ประเด็นสำคัญ

### Container Management
```bash
# ดู container ที่รันอยู่
docker ps
docker ps -a            # รวมที่หยุดแล้ว

# เริ่ม/หยุด/restart
docker start/stop/restart container_name

# ดู log
docker logs -f container_name                    # real-time
docker logs --tail 50 container_name             # 50 บรรทัดล่าสุด
docker logs --since 1h container_name            # 1 ชั่วโมงล่าสุด

# เข้าไปใน container
docker exec -it container_name bash

# ดู resource usage
docker stats
```

### Docker Compose
```bash
# รัน services จาก docker-compose.yml
docker compose up -d          # background
docker compose down           # หยุดทุก service
docker compose pull           # ดึง image ใหม่
docker compose logs -f        # ดู log ทุก service
```

### Image Management
```bash
docker images                 # ดู images ที่มี
docker pull image:tag         # ดึง image
docker rmi image_name         # ลบ image

# ทำความสะอาด
docker system prune          # ลบของที่ไม่ใช้
docker volume prune          # ลบ volumes ที่ไม่ใช้
```

### Volume & Network
```bash
docker volume ls              # ดู volumes
docker network ls             # ดู networks
docker network connect network_name container    # เชื่อม container เข้า network
```

### Docker Network

Docker Network คือ virtual network ที่ container ใช้คุยกัน — แยก isolated จาก host network และ network อื่น

```
เครื่อง NUC (192.168.1.100)
├── Docker Network "homelab" (172.20.0.0/16)
│   ├── homeassistant  → 172.20.0.2
│   ├── adguardhome    → 172.20.0.3
│   ├── nginx-proxy    → 172.20.0.4
│   ├── n8n            → 172.20.0.5
│   ├── postgres       → 172.20.0.6
│   └── agent          → 172.20.0.7
```

**Container ใน network เดียวกัน** คุยกันได้โดยตรงผ่านชื่อ container:
```yaml
# ใน docker-compose ของ n8n
environment:
  DATABASE_URL: postgresql://user:pass@postgres:5432/db
  #                                        ↑
  #                               ใช้ชื่อ container แทน IP ได้เลย
```

**Container ต่าง network** = คุยกันไม่ได้โดยตรง ✅ (isolation)

Docker ใช้ Private IP range `172.16.0.0–172.31.255.255` สำหรับ internal networks

## ตัวอย่าง / กรณีศึกษา

### docker-compose.yml Example
```yaml
services:
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    restart: unless-stopped
    volumes:
      - ./config:/config
      - /etc/localtime:/etc/localtime:ro
    environment:
      - TZ=Asia/Bangkok
    networks:
      - homelab

networks:
  homelab:
    external: true
```

## Best Practices

### Restart Policy
ทุก container ควรมี `restart: unless-stopped` เพื่อให้ restart อัตโนมัติหลัง reboot

### Resource Limits
```yaml
services:
  netdata:
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
```

### Security
- **อย่า run ด้วย root user** — ใช้ USER instruction ใน Dockerfile
- **อย่า expose database port** ออกนอก — ให้อยู่ใน Docker network เท่านั่น
- **ใช้ .env file** สำหรับ secrets — ไม่ hardcode ใน compose file
- **Scan images** ด้วย `docker scan` หรือ tools อื่น

## Gotchas

❌ **อย่าลบ volume โดยไม่ backup ก่อน** — `docker compose down -v` จะลบข้อมูลทั้งหมด

❌ **อย่า update ทุกอย่างพร้อมกัน** — update ทีละ service แล้วดู log ก่อน

❌ **อย่าใช้ latest tag ใน production** — ใช้ specific version (เช่น `:stable`, `:2024.01.01`)

## Performance Tips

- **ตั้ง vm.swappiness = 10** — ใช้ RAM ก่อน swap
- **จำกัด resource containers** ที่ไม่สำคัญ — `docker update --memory="512m" --cpus="0.5"`
- **ใช้ multi-stage build** — ลดขนาด image

## Monitoring

```bash
# ดู resource usage ทั้งหมด
docker stats --no-stream

# Script เตือน container หยุด
STOPPED=$(docker ps -a --filter "status=exited" --format "{{.Names}}")
[ -n "$STOPPED" ] && echo "Alert: Containers stopped: $STOPPED"
```

## เปรียบเทียบกับ VM

| Aspect | Docker Container | Virtual Machine |
|--------|------------------|-----------------|
| OS | แชร์ host kernel | OS แยกทั้งตัว |
| Size | MBs | GBs |
| Startup | วินาที | นาที |
| Performance | Near-native | Overhead จาก hypervisor |
| Isolation | Process-level | OS-level |

## แหล่งเรียนรู้เพิ่มเติม

- Docker Official Documentation: docs.docker.com/get-started
- TechWorld with Nana (YouTube) — อธิบายดีมาก
- Awesome Docker: github.com/veggiemonk/awesome-docker
