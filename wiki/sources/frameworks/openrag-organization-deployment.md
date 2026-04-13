---
title: "OpenRAG — Organization Deployment Guide"
type: source
source_file: raw/notes/openrag/docs-lean/openrag-organization-guide.md
published: 2026-03-17
tags: [openrag, deployment, organization, access-control, maintenance, onboarding]
related: [wiki/concepts/openrag-platform.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/frameworks/openrag/docs-lean/openrag-organization-guide.md|Original file]]
> **Additional**: [[../../../raw/notes/frameworks/openrag/docs-lean/org-rag-guide.md|Org RAG Guide]]

## สรุป

คู่มือการ deploy และใช้งาน OpenRAG ในองค์กร — setup steps, access control patterns, maintenance schedule, และ use case examples

## ประเด็นสำคัญ

### 3 ขั้นตอนหลักหลัง docker compose up

```
docker compose up -d
  ↓ ยังใช้ไม่ได้
Step 1: ตั้งค่า .env (LLM API Key, OpenSearch password, OAuth)
Step 2: Onboarding ผ่าน UI (เลือก LLM + Embedding model)
Step 3: นำเอกสารองค์กรเข้า (Ingest)
  ↓ ใช้งานได้แล้ว
```

### .env ที่จำเป็น

```env
# OpenSearch (ต้องเปลี่ยน default!)
OPENSEARCH_PASSWORD=MyOrg@Secure2024!

# LLM Provider (เลือก 1 อัน)
OPENAI_API_KEY=sk-proj-...
# หรือ ANTHROPIC_API_KEY, WATSONX_*, OLLAMA_ENDPOINT

# OAuth (ถ้าต้องการ login)
GOOGLE_OAUTH_CLIENT_ID=xxx.apps.googleusercontent.com
GOOGLE_OAUTH_CLIENT_SECRET=GOCSPX-...
```

ถ้าไม่ตั้ง OAuth → **No-auth mode** (anonymous@localhost, ทุกคนเห็นทุกอย่าง)

### config.yaml (สร้างอัตโนมัติหลัง Onboarding)

```yaml
knowledge:
  embedding_model: "text-embedding-3-small"
  chunk_size: 1000
  chunk_overlap: 200
  table_structure: true
  ocr: false

agent:
  llm_model: "gpt-4o"
  system_prompt: "You are a helpful assistant..."
```

### Access Control Patterns

```
Document ACL fields:
  owner: "hr@company.com"
  allowed_users: ["ceo@company.com"]
  allowed_groups: ["hr", "all_employees"]
```

**ตัวอย่าง: บริษัทประกัน**

| กลุ่มเอกสาร | allowed_groups |
|-------------|---------------|
| นโยบายบริษัท | all_employees |
| Product Manual | agents, supervisors |
| Loss Ratio Report | management, executives |
| Salary Grade | hr |

**Query filtering อัตโนมัติ:**
- engineer (groups: engineering, all_employees) → เห็นแค่ all_employees docs
- finance (groups: finance, all_employees) → เห็น finance + all_employees docs
- hr (groups: hr, all_employees) → เห็น hr + all_employees docs

### Knowledge Filters

"กล่อง" ที่รวมเอกสารที่เกี่ยวข้องพร้อม access control:

```bash
POST /knowledge-filter
{
  "name": "HR Policies 2024",
  "query_data": "นโยบาย HR กฎระเบียบ การลา",
  "allowed_groups": ["all_employees"]
}
```

### Maintenance Schedule

**Day-to-Day:**
- Upload เอกสารใหม่ (Content Owner แต่ละแผนก)
- ตรวจสอบ Task Queue ถ้า ingest ล้มเหลว

**Monthly:**
- Update เอกสาร, ลบ outdated
- ตรวจสอบ OpenSearch health (disk, index size)
- Backup `opensearch-data/` volume
- Review Access Control

**ไม่ต้องทำ (เลย):**
- ไม่ต้อง retrain model
- ไม่ต้อง maintain ML pipeline
- OpenSearch index ดูแลตัวเองได้

### Team Structure

| ขนาดองค์กร | Team |
|------------|------|
| < 100 คน | 1 IT Admin (part-time) + Content Owners แต่ละแผนก |
| 100-500 คน | 1 DevOps Admin + 1-2 Knowledge Managers + Content Owners |
| > 500 คน | DevOps Team + AI Engineer + Knowledge Management Team |

### Go-Live Checklist

- [ ] เปลี่ยน `OPENSEARCH_PASSWORD` จาก default
- [ ] เปลี่ยน `LANGFLOW_SUPERUSER_PASSWORD` จาก default
- [ ] ตั้งค่า SSL/HTTPS + domain + reverse proxy
- [ ] ตั้งค่า backup schedule
- [ ] ตัดสินใจ OAuth vs no-auth mode
- [ ] Ingest pilot documents (5-10 ไฟล์)
- [ ] ทดสอบ access control กับ test users
- [ ] กำหนด Admin + Content Owner

### 5 วิธีนำ Data เข้าระบบ

| วิธี | เหมาะกับ | ง่าย | Auto |
|------|---------|------|------|
| Web UI | ผู้ใช้ทั่วไป, ไฟล์น้อย | ✅ | ❌ |
| REST API | Developer, integration | ⚠️ | ✅ |
| Batch Script | เอกสารเยอะ ทำครั้งแรก | ⚠️ | ✅ |
| Cloud Sync (Google Drive/SharePoint/OneDrive) | เก็บใน cloud | ⚠️ | ✅ |
| URL Ingest | Web content, Wiki | ✅ | ✅ |

**Batch script (bash):**
```bash
for file in "$DOC_FOLDER"/*.pdf; do
  response=$(curl -s -X POST "$API_URL" \
    -H "Authorization: Bearer $API_KEY" \
    -F "file=@$file" -F "allowed_groups=$ALLOWED_GROUP")
  task_id=$(echo $response | python3 -c "import sys,json; print(json.load(sys.stdin)['task_id'])")
  # poll status...
done
```

### Data-Level Permissions (3 ระดับ)

```
owner:           "hr@company.com"          ← เจ้าของ (เห็นได้เสมอ)
allowed_users:   ["alice@co.com"]          ← user เฉพาะ
allowed_groups:  ["hr-team","all-employees"] ← กลุ่ม

Logic: เป็น owner OR อยู่ใน allowed_users OR อยู่ใน allowed_groups → เห็น
```

**ตัวอย่าง: บริษัท ABC 4 แผนก:**

| เอกสาร | allowed_groups | ใครเห็น |
|--------|---------------|---------|
| Company Handbook | all-employees | ทุกคน |
| Salary Structure | hr-team, management | HR + Mgmt |
| Q3 Financial | finance-team, management | Finance + Mgmt |
| Performance Review | allowed_users เฉพาะ | เฉพาะบุคคล |

**Groups ผ่าน OAuth vs API Key:**
```bash
# OAuth → groups มาจาก Google Workspace Groups (อัตโนมัติ)
# API Key → ส่ง X-User-Groups ใน header
curl -H "X-User-Id: john@co.com" \
     -H "X-User-Groups: hr-team,all-employees" \
     ...
```

### LLM Provider Selection

| Provider | ค่าใช้จ่าย | Privacy | คุณภาพ | Setup |
|---------|----------|---------|--------|-------|
| OpenAI (GPT-4o) | ต้องจ่าย | ออกนอก | ดีมาก | ง่าย |
| Claude (Anthropic) | ต้องจ่าย | ออกนอก | ดีมาก | ง่าย |
| WatsonX (IBM) | ต้องจ่าย | on-prem option | ดี | ปานกลาง |
| Ollama (Local) | ฟรี | ไม่ออก | ปานกลาง | ต้อง GPU |

- ข้อมูลไม่ sensitive → GPT-4o-mini (ถูก + เร็ว)
- ข้อมูล sensitive → Ollama + Llama 3.1
- Enterprise → IBM WatsonX (on-prem)

### Embedding Model Selection

```bash
SELECTED_EMBEDDING_MODEL=text-embedding-3-small  # default, ดีสุดราคาถูก (dim: 1536)
SELECTED_EMBEDDING_MODEL=text-embedding-3-large  # accuracy สูงกว่า (dim: 3072, แพง 2x)
SELECTED_EMBEDDING_MODEL=nomic-embed-text        # Ollama ฟรี (dim: 768)
# ⚠️ เปลี่ยน embedding model = ต้อง re-index ทั้งหมดใหม่
```

### Chunk Size Guide

```
< 500 tokens:  ค้นหาแม่นยำ แต่ขาด context → ตอบไม่ครบ
800-1200 tokens: ✅ แนะนำ (default ของ OpenRAG)
> 2000 tokens: context ครบ แต่ค้นหาไม่แม่น
```

### Production Security Checklist

```
[ ] เปลี่ยน OPENSEARCH_PASSWORD (default ไม่ปลอดภัย)
[ ] เปลี่ยน SECRET_KEY เป็น random 64+ chars
[ ] ตั้ง LANGFLOW_AUTO_LOGIN=False
[ ] ใช้ HTTPS (nginx reverse proxy)
[ ] ไม่ expose OpenSearch (9200) และ Langflow (7860) สู่ internet
[ ] Rotate API keys เป็นประจำ
[ ] ตั้ง CORS เฉพาะ domain ที่อนุญาต
```

### Backup & Recovery

```bash
# Backup OpenSearch data
docker exec opensearch curl -X PUT "https://localhost:9200/_snapshot/backup" ...

# Volumes ที่ต้อง backup:
# opensearch-data/  ← index data
# keys/             ← Langflow API key
# flows/            ← Flow definitions
# data/             ← Application data
# .env              ← Configuration
```

### Scaling (docker-compose.override.yml)

```yaml
services:
  opensearch:
    environment:
      - OPENSEARCH_JAVA_OPTS=-Xms4g -Xmx4g  # เพิ่ม RAM
  openrag-backend:
    deploy:
      replicas: 2  # backend 2 instances
  langflow:
    environment:
      - LANGFLOW_WORKERS=4
```

### Enterprise Use Cases

```
1. HR Knowledge Base → HR Policy, Employee Handbook → ทุกพนักงาน → LINE/Slack Bot
2. Legal & Compliance → กฎหมาย, สัญญา → Legal team → ค้นหาข้อกฎหมาย
3. IT Support → Technical docs, Runbooks → IT team → help support tickets
4. Sales Intelligence → Product catalog, Case studies → Sales → ตอบ prospect real-time
5. Customer Support → Product manual, FAQ → Public API → website chatbot
```

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/openrag-platform|OpenRAG Platform]]
