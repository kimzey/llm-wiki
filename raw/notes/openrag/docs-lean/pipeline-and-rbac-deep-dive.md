# Pipeline & RBAC Deep Dive
## ตอบคำถาม: Chunking, Langflow Storage, Role-Based Search, User System, Google

---

## 1. Docling ทำอะไรกันแน่? — แค่ "แปลงเอกสารให้อ่านได้"

**Docling ทำ 1 อย่าง: PDF/DOCX/Image → Markdown**

```
Input:  annual_report.pdf  (binary, มี layout, table, ภาพ)
          │
          ▼
     [DOCLING]
     - Layout analysis (DocLayNet model)
     - Table structure (TableFormer model)
     - OCR (ถ้าจำเป็น)
          │
          ▼
Output: "# Annual Report 2024\n\n## Revenue\n\n| Q1 | Q2 |..."
        (Markdown text ที่เครื่องอ่านได้ สะอาด มีโครงสร้าง)
```

**Docling ไม่ทำ:**
- ❌ ไม่ตัด Chunk
- ❌ ไม่สร้าง Embedding
- ❌ ไม่บันทึกลง OpenSearch
- ❌ ไม่รู้จักเรื่อง RAG เลย

Docling = เหมือน **"แปลภาษา"** จากรูปแบบไฟล์ → ข้อความที่อ่านได้

---

## 2. ใครตัด Chunk? — Langflow's Text Splitter

**Pipeline จริงของการ Ingest:**

```
[PDF ไฟล์]
     │
     ▼  HTTP POST http://docling:5001/convert
[DOCLING]  ← แค่แปลงเอกสาร
     │  ← return: Markdown string
     ▼
[LANGFLOW: RecursiveCharacterTextSplitter]  ← ตรงนี้แหละที่ตัด Chunk
     chunk_size: 1000
     chunk_overlap: 200
     │  ← return: ["chunk1...", "chunk2...", ...]
     ▼
[LANGFLOW: Embedding Component]
     │  ← return: [[0.12, -0.34, ...], ...]
     ▼
[LANGFLOW: OpenSearch Vector Store]
     │  ← index chunks + vectors ลง OpenSearch
     ▼
[OpenSearch Index: "documents"]  ← เก็บถาวรอยู่ที่นี่
```

**สรุป:**
| Component | หน้าที่ | ข้อมูลอยู่ที่ไหนหลังจากทำงาน |
|-----------|---------|--------------------------|
| Docling | PDF → Markdown | ไม่เก็บอะไรเลย (stateless) |
| Langflow | ควบคุม pipeline, ตัด chunk | ไม่เก็บ text (ชั่วคราว) |
| OpenSearch | เก็บ chunks + vectors | **เก็บถาวรอยู่ที่นี่** |

---

## 3. ไฟล์ที่ Upload ไป Langflow — จะใหญ่ไหม? ลบเองหรือเปล่า?

### ทำไมต้อง Upload ไฟล์ไป Langflow?

```
Langflow ต้องการอ่านไฟล์โดยตรงเพื่อส่งให้ Docling
ดังนั้นต้อง Upload ไปไว้ใน Langflow's temp storage ก่อน
```

### Flow จริง:

```
[Backend]
  1. รับไฟล์จาก User
  2. POST /api/v2/files  →  Langflow รับไฟล์ บันทึกใน temp storage
     Response: { "id": "file-uuid-123", "path": "/tmp/langflow/..." }
  3. POST /api/v1/run/{flow_id}  →  trigger ingestion flow
     Langflow อ่านไฟล์ → ส่ง Docling → ประมวลผล → index ใน OpenSearch
  4. DELETE /api/v2/files/file-uuid-123  →  ลบไฟล์ออกจาก Langflow
```

### ลบอัตโนมัติหลัง Ingest เสร็จ

```python
# src/services/langflow_file_service.py

async def upload_and_ingest_file(
    self,
    ...
    delete_after_ingest: bool = True  # ← default: ลบเสมอ
):
    file_id = await self.upload_user_file(file)  # upload
    await self.run_ingestion_flow(file_id, ...)  # ingest

    if delete_after_ingest and file_id:
        await self.delete_user_file(file_id)     # ลบทันที!
```

### ดังนั้น Langflow Storage จะ **ไม่โต** เพราะ:

```
ไฟล์ PDF 10MB
    → Upload ไป Langflow: 10MB (ชั่วคราว)
    → Ingest เสร็จ: Langflow ลบไฟล์ทิ้งทันที → 0MB ใน Langflow
    → OpenSearch เก็บแค่ text chunks + vectors:
      ทั่วไป PDF 10MB → OpenSearch ~2-5MB

ถึงมีเอกสาร 10,000 ไฟล์:
    Langflow storage = ไม่เพิ่ม (ลบหลัง ingest)
    OpenSearch = โตตามจำนวน chunks (ประมาณ 1-3GB/10,000 ไฟล์)
```

### พื้นที่ที่ต้อง Plan จริงๆ คือ OpenSearch:

```
ประมาณการ OpenSearch storage:
  PDF 1 ไฟล์ 10MB (20 หน้า) → ~40 chunks → ~2MB ใน OpenSearch

  100 ไฟล์   → ~200MB
  1,000 ไฟล์ → ~2GB
  10,000 ไฟล์ → ~20GB

ปรับใน docker-compose.yml:
  volumes:
    - opensearch-data:/usr/share/opensearch/data  ← mount ไปที่ disk ใหญ่พอ
```

---

## 4. Role-Based Search — ทำอย่างไร?

### ระบบ Access Control ทำงานอัตโนมัติแล้ว

OpenRAG ใช้ **Document-Level Security** โดยทุก chunk ใน OpenSearch มี:

```json
{
  "text": "เนื้อหา chunk...",
  "embeddings": [...],
  "owner": "uploader@company.com",
  "allowed_users": ["alice@company.com", "bob@company.com"],
  "allowed_groups": ["hr-team", "management"]
}
```

เมื่อ User ค้นหา → Backend **เพิ่ม filter อัตโนมัติ**:
```python
# src/utils/opensearch_queries.py
filter = {
  "bool": {
    "should": [
      {"term": {"owner": current_user_email}},
      {"term": {"allowed_users": current_user_email}},
      {"terms": {"allowed_groups": user_groups}}
    ],
    "minimum_should_match": 1
  }
}
```

**User เห็นแค่เอกสารที่ตัวเองมีสิทธิ์** — ไม่มีทางข้าม filter นี้ได้

---

### วิธีตั้งค่า RBAC ให้สอดคล้องกับโครงสร้างองค์กร

#### แนวทางที่ 1: Manual Groups (ง่ายที่สุด)

```bash
# ตั้งค่า allowed_groups ตอน ingest โดยตรง
# โดยตั้งชื่อ group ให้ตรงกับแผนก/บทบาทที่มีใน JWT token

# HR document — เฉพาะ HR และ Management เห็น
curl -X POST "http://localhost:8080/v1/documents/ingest" \
  -H "Authorization: Bearer API_KEY" \
  -F "file=@salary_structure.pdf" \
  -F "allowed_groups=hr-team,management"

# Company handbook — ทุกคนเห็น
curl -X POST "http://localhost:8080/v1/documents/ingest" \
  -H "Authorization: Bearer API_KEY" \
  -F "file=@company_handbook.pdf" \
  -F "allowed_groups=all-employees"

# Technical spec — เฉพาะ Engineering
curl -X POST "http://localhost:8080/v1/documents/ingest" \
  -H "Authorization: Bearer API_KEY" \
  -F "file=@api_spec.pdf" \
  -F "allowed_groups=engineering-team"
```

#### แนวทางที่ 2: ใช้ Google Drive Permissions (อัตโนมัติ)

ถ้าใช้ **Google Drive Connector** ระบบจะดึง permission จาก Google Drive มาเป็น ACL อัตโนมัติ:

```
Google Drive: "salary_structure.pdf"
  Shared with:
    - hr@googlegroups.com (group)
    - ceo@company.com (user)
    - management@googlegroups.com (group)

→ OpenRAG ingest → ACL:
  allowed_users: ["ceo@company.com"]
  allowed_groups: ["hr@googlegroups.com", "management@googlegroups.com"]
```

**ข้อดี:** ไม่ต้องตั้งค่า permissions ซ้ำ — ใช้จาก Drive เลย
**ข้อจำกัด:** ดึงได้เฉพาะ permission ที่ share ไว้ใน Google Drive เท่านั้น (ไม่ใช่ Org Chart จาก Google Admin)

---

### ข้อมูล User ที่ต้องมีสำหรับ RBAC

เวลา User ค้นหา ระบบต้องรู้ 2 อย่าง:
1. **email** ของ User (รู้จาก JWT token หลัง login)
2. **groups** ที่ User อยู่ (ต้องส่งมาใน JWT หรือ header)

```python
# JWT Token ที่ OpenRAG สร้างตอน login มี:
{
  "email": "john@company.com",
  "name": "John Smith",
  "user_id": "google-uid-123",
  "roles": ["openrag_user"]
  # ⚠️ ไม่มี groups อัตโนมัติ — ดูหัวข้อถัดไป
}
```

---

## 5. User System — Groups/Roles มาจากไหน?

### ตอนนี้ระบบดึงข้อมูล User อะไรบ้าง?

```
Google Login (OAuth) ดึงมาได้:
  ✅ email     (john@company.com)
  ✅ name      (John Smith)
  ✅ picture   (profile photo URL)
  ✅ user_id   (Google unique ID)

  ❌ department (แผนก)
  ❌ job_title (ตำแหน่ง)
  ❌ Google Groups membership (กลุ่มที่อยู่)
  ❌ Org Unit (สายงาน)
```

**ทำไมถึงดึง Groups ไม่ได้อัตโนมัติ?**
- Google OAuth standard scope (`email`, `profile`) ไม่รวม group membership
- ต้องใช้ **Google Admin SDK** (`admin.googleapis.com`) แยกต่างหาก
- ต้องมี Service Account + Admin permission พิเศษ

### 3 วิธีแก้ปัญหา Groups

---

#### วิธีที่ A: ส่ง Groups จาก App ที่ Integrate (แนะนำสำหรับ API integration)

ถ้าคุณมี App ที่รู้ว่า User มี Role อะไร → ส่งมาใน header:

```bash
# App ของคุณรู้ว่า john เป็น hr-team + all-employees
curl -X POST "http://localhost:8080/v1/chat" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "X-User-Id: john@company.com" \
  -H "X-User-Groups: hr-team,all-employees" \
  -d '{"message": "ลาป่วยได้กี่วัน?"}'

# ระบบจะค้นหาแค่เอกสารที่ hr-team หรือ all-employees เข้าถึงได้
```

---

#### วิธีที่ B: Google Drive Connector (สำหรับ Drive Sync)

ใช้ Google Drive Connector ซึ่งดึง permissions จากไฟล์ใน Drive มาเป็น ACL:

```
1. Share ไฟล์ใน Google Drive ไปที่ Google Groups
   - salary.pdf → shared with "hr@yourcompany.com" (Google Group)
   - handbook.pdf → shared with "all@yourcompany.com" (Google Group)

2. OpenRAG Sync จาก Drive → ดึง Google Group permissions มาเป็น allowed_groups

3. ตอน User ค้นหา → ระบบ filter ด้วย Google Group emails
```

**การตั้งค่า:**
```bash
# .env
GOOGLE_OAUTH_CLIENT_ID=your-client-id
GOOGLE_OAUTH_CLIENT_SECRET=your-client-secret

# แล้ว connect ผ่าน UI หรือ API
POST /connectors/google_drive/sync
```

---

#### วิธีที่ C: Custom JWT — Inject Groups ตอน Login (ยืดหยุ่นที่สุด)

ถ้าบริษัทมี **Identity Provider (IdP)** เช่น Okta, Azure AD, หรือ Google Workspace Admin → สร้าง JWT ที่มี groups อยู่แล้ว แล้วให้ OpenRAG ใช้ JWT นั้น

```python
# ตัวอย่าง: Backend ของคุณสร้าง JWT ที่มี groups
# หลังจาก Google Login → ดึง groups จาก Google Admin API เพิ่ม

import jwt
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build

async def get_user_groups_from_google_admin(user_email: str, admin_creds):
    """ดึง Google Groups ที่ user เป็นสมาชิก"""
    service = build("admin", "directory_v1", credentials=admin_creds)

    groups_result = service.groups().list(
        userKey=user_email,
        domain="yourcompany.com"
    ).execute()

    return [g["email"] for g in groups_result.get("groups", [])]

# สร้าง JWT พร้อม groups
async def create_enriched_jwt(user_email: str):
    groups = await get_user_groups_from_google_admin(user_email, admin_creds)
    # groups = ["hr@yourcompany.com", "all-employees@yourcompany.com"]

    token = jwt.encode({
        "email": user_email,
        "groups": groups,  # ← groups จาก Google Admin
        "exp": ...,
    }, SECRET_KEY)

    return token
```

**ข้อกำหนด Google Admin API:**
```
ต้องใช้:
  - Google Workspace Admin account
  - Enable Admin SDK API ใน Google Cloud Console
  - Service Account ที่มี Domain-Wide Delegation
  - Scope: https://www.googleapis.com/auth/admin.directory.group.readonly
```

---

## 6. Google Workspace Integration — ต่อได้เลยไหม?

### บริษัทใช้ Google Account ทุกคน → ต่อได้ ทำยังไง?

#### Step 1: Setup Google OAuth ใน Google Cloud Console

```
1. ไป https://console.cloud.google.com
2. Create/เลือก Project
3. APIs & Services → OAuth consent screen
   - User type: Internal (เฉพาะ @yourcompany.com)
   - App name: "Our Company RAG"
   - Scopes: openid, email, profile
4. APIs & Services → Credentials
   - Create Credentials → OAuth 2.0 Client IDs
   - Application type: Web application
   - Authorized redirect URIs: http://your-openrag/auth/callback
5. Copy Client ID และ Client Secret
```

#### Step 2: ตั้งค่าใน OpenRAG

```bash
# .env
GOOGLE_OAUTH_CLIENT_ID=123456789-abc.apps.googleusercontent.com
GOOGLE_OAUTH_CLIENT_SECRET=GOCSPX-xxxxxxxxxxxxxxx
```

#### Step 3: User Login Flow

```
User เปิด http://your-openrag.internal
    → คลิก "Sign in with Google"
    → Redirect ไป Google (เลือก @yourcompany.com account)
    → Google ขอ permission: profile + email
    → User อนุมัติ
    → Redirect กลับ http://your-openrag.internal/auth/callback
    → OpenRAG สร้าง JWT token (email, name, picture)
    → User เข้าใช้งานได้

ข้อมูลที่ได้:
  ✅ email: john.smith@yourcompany.com
  ✅ name: John Smith
  ✅ picture: https://lh3.googleusercontent.com/...
```

#### Step 4: ตั้งค่า Groups สำหรับ RBAC

**Option A (ง่ายที่สุด): Google Drive Sharing**
```
วิธีทำงาน:
  1. สร้าง Google Groups ตามแผนก:
     - all-employees@yourcompany.com
     - hr-team@yourcompany.com
     - finance-team@yourcompany.com
     - engineering@yourcompany.com

  2. Share เอกสารใน Google Drive ตาม Group:
     - HR Policy → share กับ all-employees@yourcompany.com
     - Salary Data → share กับ hr-team@yourcompany.com

  3. Connect Google Drive ใน OpenRAG
  4. Sync → ACL ถูก map จาก Drive permissions อัตโนมัติ

  ผล: User ที่เป็นสมาชิก hr-team@yourcompany.com
       → ค้นหาเจอเอกสาร Salary Data
       User ที่ไม่ใช่สมาชิก → ค้นหาไม่เจอ ✅
```

**Option B (ยืดหยุ่น): Manual Group ตอน Ingest**
```
ตั้ง allowed_groups ให้ตรงกับ Google Group emails:

curl -X POST "http://localhost:8080/v1/documents/ingest" \
  -F "file=@hr_policy.pdf" \
  -F "allowed_groups=all-employees@yourcompany.com"

curl -X POST "http://localhost:8080/v1/documents/ingest" \
  -F "file=@salary.pdf" \
  -F "allowed_groups=hr-team@yourcompany.com"

แต่ตอน User ค้นหา → ต้องส่ง groups ที่ user อยู่มาด้วย
(ดูวิธีที่ A และ C ด้านบน)
```

---

## 7. สรุปภาพรวม — ระบบ User ทั้งหมด

```
┌──────────────────────────────────────────────────────────────────┐
│                     User Identity Flow                            │
│                                                                  │
│  [User] → Login with Google                                      │
│               │                                                  │
│               ▼ Google OAuth                                     │
│         email, name, picture                                     │
│         (ไม่มี groups)                                          │
│               │                                                  │
│               ▼                                                  │
│  [OpenRAG] สร้าง JWT token                                       │
│               │                                                  │
│               ▼                                                  │
│  [API Request] มี JWT                                            │
│               │                                                  │
│         ┌─────┴──────┐                                          │
│         │             │                                          │
│    เป็น email      เป็น groups                                  │
│    → filter by      → filter by                                 │
│      allowed_users    allowed_groups                            │
│         │             │                                          │
│         └─────┬────────┘                                         │
│               │                                                  │
│  [OpenSearch] return เฉพาะ docs ที่ user มีสิทธิ์              │
│                                                                  │
│  ปัญหา: Groups ต้องมาจากไหน?                                    │
│    Option A: ส่ง groups มาใน API header (จาก app ของคุณ)       │
│    Option B: Google Drive Connector (permissions จาก Drive)     │
│    Option C: Google Admin SDK + custom JWT enrichment           │
└──────────────────────────────────────────────────────────────────┘
```

---

## 8. ตัวอย่าง Scenario จริง — บริษัทใช้ Google Workspace

### Setup ครั้งแรก:

```
1. สร้าง Google Groups ใน Google Admin Console (admin.google.com):
   all-employees@yourco.com  → add ทุกคน
   hr-team@yourco.com        → add HR staff
   finance@yourco.com        → add Finance staff
   engineering@yourco.com    → add engineers
   management@yourco.com     → add managers

2. ตั้งค่า OpenRAG:
   GOOGLE_OAUTH_CLIENT_ID=...
   GOOGLE_OAUTH_CLIENT_SECRET=...

3. Share เอกสารใน Google Drive ตาม Group
   แล้ว Sync ด้วย Google Drive Connector
```

### ทุกวัน:

```
1. เพิ่มพนักงานใหม่:
   → Add เข้า all-employees@yourco.com ใน Google Admin
   → พนักงานใหม่ค้นหาเอกสาร all-employees ได้ทันที (ถ้า groups sync)

2. อัปโหลดเอกสารใหม่:
   → Share ใน Google Drive → Sync → ACL อัปเดตอัตโนมัติ

3. User Login:
   → Sign in with Google @yourco.com
   → ค้นหาได้เฉพาะเอกสารที่ตัวเองมีสิทธิ์
```

---

## 9. Checklist สำหรับ RBAC Setup

```
[ ] ตัดสินใจวิธี Groups:
    [ ] A: Manual groups ใน API (ingest + query)
    [ ] B: Google Drive Connector + Drive sharing
    [ ] C: Google Admin SDK integration (ต้องพัฒนาเพิ่ม)

[ ] กำหนด Group naming convention ให้ชัด:
    เช่น: {dept}-team@yourco.com

[ ] ทดสอบ access control:
    [ ] Login เป็น HR → เห็นเอกสาร HR
    [ ] Login เป็น Finance → ไม่เห็นเอกสาร HR
    [ ] Login เป็น CEO → เห็นทุกอย่าง (อยู่ใน management group)

[ ] กำหนด fallback:
    ถ้า user ไม่มี group ใดเลย → เห็นแค่ public docs (allowed_groups=all-employees)
```

---

*กลับไป: [org-rag-guide.md](./org-rag-guide.md)*
*ดู Index: [README.md](./README.md)*
