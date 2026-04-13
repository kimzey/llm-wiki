# 🌐 พื้นฐาน Network & Infrastructure
### สำหรับคนที่ไม่รู้อะไรเลย — อธิบายตั้งแต่ศูนย์

---

## 1. บ้านคุณเชื่อมอินเทอร์เน็ตยังไง?

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

**Router** ทำหน้าที่เป็น "ประตูบ้าน" — แปลง IP สาธารณะ 1 อัน
ให้กระจายเป็น IP ส่วนตัวหลายอัน (192.168.x.x) ให้ทุก device ในบ้าน

---

## 2. IP Address คืออะไร?

คิดง่ายๆ เหมือน "ที่อยู่บ้าน"

### IP มี 2 ประเภท

```
┌─────────────────────────────────────────────────┐
│  Public IP (IP สาธารณะ)                         │
│  → ที่อยู่ "บ้านทั้งหลัง" บนอินเทอร์เน็ต        │
│  → ทุกคนในโลกเห็นได้                             │
│  → ตัวอย่าง: 101.51.147.23                      │
│  → ISP กำหนดให้ เปลี่ยนได้ตลอด (Dynamic IP)     │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  Private IP (IP ส่วนตัว)                        │
│  → ที่อยู่ "ห้องในบ้าน" แต่ละ device           │
│  → คนนอกบ้านมองไม่เห็น                          │
│  → ตัวอย่าง: 192.168.1.100 (NUC ของคุณ)        │
│                192.168.1.101 (มือถือ)           │
│  → Router กำหนดให้                              │
└─────────────────────────────────────────────────┘
```

### ช่วง IP ที่เป็น Private เสมอ (ไม่มีในอินเทอร์เน็ต)
```
192.168.0.0   →  192.168.255.255   (บ้านทั่วไป)
10.0.0.0      →  10.255.255.255    (องค์กร)
172.16.0.0    →  172.31.255.255    (Docker ใช้)
```

---

## 3. Port คืออะไร?

ถ้า IP = "บ้าน" → Port = "ประตูห้องต่างๆ ในบ้าน"

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

**Port ที่ปิดอยู่** = ไม่มีใครเข้าได้แม้รู้ IP
**Port ที่เปิดอยู่** = มีโปรแกรมรอรับการเชื่อมต่อ

Port ที่รู้จักกันทั่วไป (Well-known ports):
```
21   → FTP
22   → SSH
25   → Email (SMTP)
53   → DNS
80   → HTTP
443  → HTTPS
3306 → MySQL
5432 → PostgreSQL
```

---

## 4. DNS คืออะไร?

คอมพิวเตอร์คุยกันด้วย IP แต่คนจำ IP ไม่ได้
DNS เลยเป็น "สมุดโทรศัพท์" แปลง domain เป็น IP

```
คุณพิมพ์: google.com
     ↓
DNS Server ค้นหา
     ↓
ได้ IP: 142.250.185.46
     ↓
Browser เชื่อมต่อไปที่ IP นั้น
```

### DNS ของ Home Server คุณ
```
ha.yourdomain.com  →  (ชี้ไปที่ Cloudflare Tunnel)
                          ↓
                    Cloudflare รู้ว่าต้องส่งต่อไปที่ NUC
                          ↓
                    192.168.1.100:8123
```

---

## 5. NAT คืออะไร? (ทำไมคนนอกบ้านเข้าไม่ได้)

**NAT = Network Address Translation**
Router แปลง IP เวลาส่งข้อมูลออกนอก

```
NUC (192.168.1.100) ส่ง request ออกไป Google
           ↓
Router รับ → เปลี่ยน IP เป็น Public IP บ้าน (101.51.147.23)
           ↓
Google เห็นแค่ IP บ้าน ไม่รู้ว่า device ไหนใน
           ↓
Google ตอบกลับมาที่ 101.51.147.23
           ↓
Router รู้ว่าต้องส่งต่อให้ 192.168.1.100
```

**ปัญหา:** ถ้าคนนอกส่งมาหา 101.51.147.23 โดยตรง
Router ไม่รู้ว่าจะส่งต่อให้ device ไหน → ทิ้งทันที

นี่คือเหตุผลที่ **คนนอกบ้านเข้า Server ไม่ได้โดยตรง** ✅

---

## 6. Port Forwarding คืออะไร?

บอก Router ว่า "ถ้ามีคนมาที่ port นี้ ส่งต่อให้ device นี้เลย"

```
คนนอกบ้าน → 101.51.147.23:8123
                   ↓
Router เห็น port 8123 → ส่งต่อให้ 192.168.1.100:8123
                   ↓
Home Assistant ตอบกลับ ✅
```

### ⚠️ ความเสี่ยงของ Port Forwarding

```
อินเทอร์เน็ต
     │
     ▼
101.51.147.23:22  ← Bot สแกนทั่วโลกตลอดเวลา!
     │               พยายาม brute force password
     ▼
NUC ของคุณ  ← ถ้า password อ่อนหรือมีช่องโหว่ = โดนแฮก
```

**ของที่อันตรายถ้าเปิด Port:**
- Port 22 (SSH) — โดนสแกนและ brute force ตลอด
- Port 3389 (RDP) — อันตรายมาก
- Database ports (3306, 5432) — ห้ามเปิดเด็ดขาด

---

## 7. วิธีเข้า Server จากนอกบ้าน (3 แบบ)

### แบบที่ 1 — Port Forwarding (เก่า, เสี่ยง)
```
คุณ (นอกบ้าน) → Public IP:Port → Router → NUC

ข้อดี:  ง่าย ตรงไปตรงมา
ข้อเสีย: IP บ้านเปิดเผย, โดนสแกน, เสี่ยงโดนแฮก
ใช้เมื่อ: ไม่มีทางเลือกอื่น หรือ dev ใน local
```

### แบบที่ 2 — VPN (ปลอดภัย, แนะนำ)
```
คุณ (นอกบ้าน) → Tailscale Network (encrypted tunnel)
                        ↓
                   NUC (เหมือนอยู่ในบ้าน)

ข้อดี:  ปลอดภัยมาก, IP บ้านไม่เปิด, ใช้ง่าย
ข้อเสีย: ต้องติด Tailscale ทุก device ที่ใช้
ใช้เมื่อ: SSH, จัดการ server, dev work ส่วนตัว
```

### แบบที่ 3 — Reverse Tunnel (ทันสมัย, ปลอดภัยสุด)
```
NUC → เชื่อมออกไปหา Cloudflare (outbound)
              ↓
คุณ/ผู้ใช้ → yourdomain.com → Cloudflare
              ↓ (ส่งต่อผ่าน Tunnel ที่ NUC เปิดไว้)
              NUC

ข้อดี:  ไม่เปิด port เลย, IP ซ่อน, SSL ฟรี
        ใครๆ เข้าได้ผ่าน domain (ถ้าอยากแชร์)
ข้อเสีย: ผ่าน Cloudflare (third-party)
ใช้เมื่อ: Web services, API, ให้คนอื่นเข้าด้วย
```

---

## 8. Tailscale ทำงานยังไง?

```
NUC (ในบ้าน)                    Laptop (นอกบ้าน)
192.168.1.100                   (IP อะไรก็ได้)
100.64.0.5 ←── Tailscale ───→ 100.64.0.2
               Mesh VPN
               (encrypted)

Tailscale Coordination Server
(แค่ช่วย handshake ครั้งแรก หลังจากนั้น
 device คุยกันตรงๆ ไม่ผ่าน server)
```

หลังจาก connect แล้ว:
```bash
ssh admin@100.64.0.5   # เหมือนอยู่ในบ้านเลย
```

**ทำไมปลอดภัย?**
- ทุก connection เข้ารหัส WireGuard (military-grade)
- ต้องล็อกอิน Google/GitHub ก่อนใช้
- คนนอกที่ไม่ได้ login account เดียวกัน เข้าไม่ได้

---

## 9. Cloudflare Tunnel ทำงานยังไง?

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

---

## 10. SSL / HTTPS คืออะไร?

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

**SSL Certificate** = ใบรับรองว่า "เว็บนี้เป็นของจริง ไม่ใช่ของปลอม"
- Let's Encrypt ออกให้ฟรี
- Cloudflare ออกให้ฟรีอัตโนมัติ

---

## 11. Reverse Proxy คืออะไร?

**ปัญหา:** มี services หลายตัว แต่ port 443 มีแค่อันเดียว

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

Nginx เป็น "พนักงานต้อนรับ" ที่รู้ว่าจะส่งต่อใครไปห้องไหน
โดยดูจาก domain name ที่คนเข้ามา

---

## 12. Docker Network ทำงานยังไง?

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

Container ใน network เดียวกันคุยกันได้โดยตรงผ่านชื่อ:
```yaml
# ใน docker-compose ของ n8n
DATABASE_URL: postgresql://user:pass@postgres:5432/db
#                                        ↑
#                               ใช้ชื่อ container แทน IP ได้เลย
```

Container ที่ไม่ได้อยู่ใน network เดียวกัน = คุยกันไม่ได้ ✅

---

## 13. Firewall (UFW) คืออะไร?

```
อินเทอร์เน็ต
     │
     ▼
[ UFW Firewall ] ← ตรวจสอบทุก connection ที่เข้า/ออก
     │
     ├── Port 22 (SSH)  → อนุญาต ✅
     ├── Port 80 (HTTP) → อนุญาต ✅
     ├── Port 443 (HTTPS) → อนุญาต ✅
     ├── Port 3306 (MySQL) → บล็อก ❌
     └── Port อื่นๆ → บล็อกหมด ❌
     │
     ▼
    NUC
```

---

## 14. ความปลอดภัยโดยรวมของ Setup นี้

```
ระดับความปลอดภัย (1-10)

Port Forwarding แบบเปิดดิบ     [██░░░░░░░░] 2/10
Port Forward + Strong Password  [████░░░░░░] 4/10
Cloudflare Tunnel               [████████░░] 8/10
Tailscale VPN                   [█████████░] 9/10
Cloudflare + Tailscale ด้วยกัน  [██████████] 10/10
```

### Plan ของคุณใช้อะไร?

```
SSH (admin ใช้)     → Tailscale VPN ✅ ปลอดภัยมาก
Web Services        → Cloudflare Tunnel ✅ ปลอดภัยมาก
Database            → ไม่เปิดออกนอกเลย ✅ ปลอดภัยสุด
Docker network      → Isolated network ✅
UFW Firewall        → บล็อก port ที่ไม่ต้องการ ✅
```

---

## 15. คำถามที่มักสงสัย

### "ถ้าใช้ Cloudflare Tunnel ใครๆ ก็เข้าได้ไหม?"

```
เข้าถึง URL ได้ ≠ ทำอะไรได้

ha.yourdomain.com เข้าได้ทุกคน
แต่ต้อง Login ด้วย username + password ก่อน
ถ้าไม่มี account → ไม่เห็นอะไรเลย ✅

เหมือนร้านค้าที่มีประตูเข้าได้
แต่ข้างในต้องมีบัตรพนักงานถึงจะใช้อะไรได้
```

### "Home Assistant ปลอดภัยพอไหมถ้าเปิดออกนอก?"

ปลอดภัยถ้า:
- ใช้ password ที่แข็งแรง (12+ ตัว)
- เปิด 2FA (Two-Factor Authentication)
- เข้าผ่าน Cloudflare (ไม่ port forward ตรงๆ)

### "ถ้าไฟดับหรือ NUC ปิด Tunnel จะขาดไหม?"

ขาดครับ แต่พอเปิดใหม่ก็กลับมาทันที
นี่คือเหตุผลที่แนะนำซื้อ UPS

### "ISP เปลี่ยน Public IP บ่อยมีผลไหม?"

ไม่มีผลเลยกับ Cloudflare Tunnel และ Tailscale
เพราะไม่ได้ใช้ Public IP ของบ้านโดยตรง ✅

---

## 16. สรุปภาพรวมทั้งหมด

```
╔══════════════════════════════════════════════════════╗
║                    อินเทอร์เน็ต                       ║
╚══════════════════════════════════════════════════════╝
         │                          │
    ┌────┴─────┐              ┌─────┴────┐
    │Cloudflare│              │Tailscale │
    │  Tunnel  │              │  Server  │
    └────┬─────┘              └─────┬────┘
         │ HTTPS (443)              │ WireGuard
         │ ทุกคนเข้าได้             │ เฉพาะ device คุณ
         │ ต้อง login               │
    ┌────┴──────────────────────────┴────┐
    │           NUC7i5BNH               │
    │         192.168.1.100             │
    │                                   │
    │  ┌──────────┐  ┌──────────────┐   │
    │  │  Docker  │  │ Claude Code  │   │
    │  │ Services │  │  AI Agent    │   │
    │  └──────────┘  └──────────────┘   │
    └───────────────────────────────────┘
         │
    ┌────┴────┐
    │ Router  │ ← ไม่ต้องเปิด Port ใดๆ ✅
    └────┬────┘
         │
    ═════╪═════ LAN บ้านคุณ
         │
    (อุปกรณ์อื่นๆ ในบ้าน)
```

---

## 17. คำศัพท์ที่ต้องรู้

| คำ | ความหมาย |
|---|---|
| IP Address | ที่อยู่ของ device บนเครือข่าย |
| Public IP | IP ที่โลกภายนอกเห็น (ของบ้านทั้งหลัง) |
| Private IP | IP ภายในบ้าน (192.168.x.x) |
| Port | ช่องทางสื่อสารของโปรแกรมต่างๆ |
| DNS | แปลง domain name เป็น IP |
| NAT | Router แปลง IP ส่วนตัว↔สาธารณะ |
| Port Forwarding | เปิดช่องให้คนนอกเข้า port ที่กำหนด |
| VPN | อุโมงค์เข้ารหัสเชื่อม device ปลอดภัย |
| Reverse Proxy | ตัวกลางรับ request แล้วส่งต่อให้ service |
| SSL/TLS | เข้ารหัส traffic ระหว่างผู้ใช้กับ server |
| HTTPS | HTTP + SSL = ปลอดภัย |
| Firewall | กำแพงบล็อก traffic ที่ไม่ต้องการ |
| Docker | รัน application ใน container แยกจากกัน |
| Container | กล่องแยกสำหรับแต่ละ application |
| Tunnel | อุโมงค์ส่งข้อมูลผ่านเครือข่ายอื่น |
| Cloudflare | บริษัทที่ให้ CDN, DNS, Tunnel ฟรี |
| Tailscale | VPN Mesh ที่ใช้งานง่าย ฟรี |
| WireGuard | protocol VPN ที่เร็วและปลอดภัยที่สุด |

---

*เอกสารนี้เป็นส่วนหนึ่งของ Home Server Setup Plan*  
*Intel NUC7i5BNH · Ubuntu 24.04 LTS · Docker*
MDEOF