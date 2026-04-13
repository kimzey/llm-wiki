---
title: "OpenRAG — Access Control & RBAC Deep Dive"
type: source
source_file: raw/notes/openrag/docs-lean/openrag-allowed-groups-google.md
published: 2026-03-18
tags: [openrag, rbac, dls, opensearch, jwt, access-control, google-drive]
related: [wiki/concepts/openrag-platform.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/openrag/docs-lean/openrag-allowed-groups-google.md|allowed_groups Google Analysis]]
> **Additional**: [[../../raw/notes/openrag/docs-lean/pipeline-and-rbac-deep-dive.md|Pipeline & RBAC Deep Dive]]

## สรุป

วิเคราะห์เชิงลึกเกี่ยวกับ Document-Level Security (DLS) ใน OpenRAG — ทั้ง `allowed_users` และ `allowed_groups` field, architectural gap ของ group enforcement, และวิธีตั้งค่า RBAC ในองค์กร

## ประเด็นสำคัญ

### OpenSearch DLS Policy (ปัจจุบัน)

```yaml
# securityconfig/roles.yml
dls: >
  {"bool":{"should":[
    {"term":{"owner":"${user.name}"}},
    {"term":{"allowed_users":"${user.name}"}},
    {"bool":{"must_not":{"exists":{"field":"owner"}}}}
  ],"minimum_should_match":1}}
```

`${user.name}` = ค่าจาก JWT claim `sub` (user UUID)

ผู้ใช้เห็นเฉพาะ doc ที่:
1. `owner == user.sub` — เป็นเจ้าของ
2. `allowed_users contains user.sub` — ถูกระบุชื่อ
3. `owner field ไม่มี` — สาธารณะ

### Architectural Gap: allowed_groups ไม่ถูก enforce

```
Google Drive Connector ตอน ingest:
  ✅ ดึง group email จาก permissions().list() API
  ✅ เก็บใน OpenSearch field: allowed_groups = ["finance@co.com"]

แต่ตอน query:
  ❌ DLS policy ไม่มี condition สำหรับ allowed_groups เลย
  ❌ JWT token ไม่มี group membership claims
  → user ที่อยู่ใน finance@co.com ก็ยังเห็นเอกสารไม่ได้!
```

**เหตุผลที่ JWT ไม่มี groups:**
- Google OAuth standard scope (`email`, `profile`) ไม่รวม group membership
- ต้องใช้ Google Directory API แยก + Service Account + Admin consent

### วิธีแก้ไข 3 แบบ

**Option A (แนะนำ): เพิ่ม groups claim ใน JWT**
1. Call Google Directory API ตอน OAuth callback → ดึง group emails
2. ใส่ `"groups": [...]` ใน JWT payload
3. อัพเดท DLS policy: เพิ่ม `{"terms":{"allowed_groups":${user.roles}}}`
- ⚠️ ต้องใช้ scope `admin.directory.group.readonly` + domain-wide delegation

**Option B: Webhook ACL Sync**
- Google Drive webhook notify เมื่อ permissions เปลี่ยน
- Backend call `batch_update_acls()` (มีใน `acl_utils.py` แล้ว!)
- Expand group → user emails แล้วเก็บใน `allowed_users` แทน
- ข้อดี: ไม่ต้องแก้ DLS; ข้อเสีย: ต้อง sync ทุกครั้ง group เปลี่ยน

**Option C: ใช้แค่ allowed_users**
- Share ไฟล์แบบ individual user ไม่ใช่ group
- ระบบทำงานได้ปกติ — เหมาะองค์กรที่ไม่ใช้ Google Workspace Groups

### RBAC Patterns ในองค์กร

**Manual Groups (ง่ายที่สุด):**
```bash
# ตั้ง allowed_groups ตาม group name ที่ตรงกับ JWT token
curl -X POST "http://localhost:8080/v1/documents/ingest" \
  -F "file=@salary_structure.pdf" \
  -F "allowed_groups=hr-team,management"
```

**Google Drive Connector (อัตโนมัติ):**
```
Share ไฟล์ใน Drive → Google Groups
→ Connector ดึง permissions มาเป็น ACL อัตโนมัติ
→ (แต่ enforce ได้แค่ allowed_users ไม่ใช่ groups จนกว่าจะแก้ DLS)
```

**Via API Header (สำหรับ custom integration):**
```bash
# App ที่รู้ว่า user มี groups → ส่งมาใน header
curl -X POST "http://localhost:8080/v1/chat" \
  -H "X-User-Id: john@company.com" \
  -H "X-User-Groups: hr-team,all-employees" \
  -d '{"message": "ลาป่วยได้กี่วัน?"}'
```

### สรุป: อะไรทำงาน อะไรไม่ทำงาน

| Feature | สถานะ |
|---------|--------|
| เก็บ allowed_users ตอน ingest | ✅ ทำงาน |
| เก็บ allowed_groups ตอน ingest | ✅ ทำงาน (เก็บ field ได้) |
| enforce owner check ที่ query time | ✅ ทำงาน |
| enforce allowed_users check | ⚠️ บางส่วน (UUID vs email mismatch possible) |
| enforce allowed_groups check | ❌ ไม่ทำงาน (DLS ไม่มี condition นี้) |
| JWT มี group membership | ❌ ไม่มี |

### Langflow Permission Architecture

```
Langflow Flow เอง → ❌ ไม่มี permission checking ในตัวเลย
Permission จริงอยู่ที่:
  ✅ OpenSearch + JWT (OIDC validation)
  ✅ Document ACL fields: owner, allowed_users, allowed_groups
  ✅ KNN search auto-filter ตาม user identity
```

ถ้า bypass Backend ไปหา Langflow โดยตรง:
- ถ้าไม่ส่ง JWT → OpenSearch reject (ถ้า security enabled)
- ถ้าส่ง JWT ถูก + bypass Backend → ACL ยังทำงาน แต่ Rate limit/Audit log ถูก bypass

### Custom JWT Enrichment — Option A (รายละเอียด)

```python
# ดึง Google Groups จาก Google Admin API หลัง OAuth callback
async def get_user_groups_from_google_admin(user_email: str, admin_creds):
    service = build("admin", "directory_v1", credentials=admin_creds)
    groups_result = service.groups().list(
        userKey=user_email, domain="yourcompany.com"
    ).execute()
    return [g["email"] for g in groups_result.get("groups", [])]

async def create_enriched_jwt(user_email: str):
    groups = await get_user_groups_from_google_admin(user_email, admin_creds)
    token = jwt.encode({
        "email": user_email,
        "groups": groups,  # ← ["hr@yourco.com", "all-employees@yourco.com"]
        "exp": ...,
    }, SECRET_KEY)
    return token
```

**Google Admin API Requirements:**
```
- Google Workspace Admin account
- Enable Admin SDK API ใน Google Cloud Console
- Service Account ที่มี Domain-Wide Delegation
- Scope: https://www.googleapis.com/auth/admin.directory.group.readonly
```

### Google Workspace Integration — Step by Step

**Step 1: Setup OAuth ใน Google Cloud Console:**
```
APIs & Services → OAuth consent screen (Internal)
Scopes: openid, email, profile
Credentials → OAuth 2.0 Client IDs (Web app)
Authorized redirect URIs: http://your-openrag/auth/callback
```

**Step 2: .env**
```env
GOOGLE_OAUTH_CLIENT_ID=123456789-abc.apps.googleusercontent.com
GOOGLE_OAUTH_CLIENT_SECRET=GOCSPX-xxxxxxxxxxxxxxx
```

**Step 3: กำหนด Groups สำหรับ RBAC**

*Option A — Google Drive Sharing (ง่ายที่สุด):*
```
1. สร้าง Google Groups: all-employees@, hr-team@, finance@, engineering@, management@
2. Share เอกสารใน Drive ตาม Group
3. Connect Google Drive Connector ใน OpenRAG → Sync
→ ACL ถูก map จาก Drive permissions อัตโนมัติ
```

*Option B — Manual ตอน Ingest:*
```bash
curl -X POST ".../v1/documents/ingest" \
  -F "allowed_groups=all-employees@yourco.com"
# แต่ต้องส่ง groups ที่ user อยู่มาด้วยตอน query
```

**RBAC Setup Checklist:**
```
[ ] ตัดสินใจวิธี Groups:
    [ ] A: JWT enrichment (Google Admin API)
    [ ] B: Google Drive Connector + Drive sharing
    [ ] C: Manual groups ใน API header
[ ] กำหนด Group naming convention: {dept}-team@yourco.com
[ ] ทดสอบ access control:
    [ ] HR login → เห็นเอกสาร HR
    [ ] Finance login → ไม่เห็นเอกสาร HR
[ ] กำหนด fallback: ถ้า user ไม่มี group → เห็นแค่ public docs
```

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

- เอกสาร phase3-opensearch.md อธิบายว่า allowed_groups ทำงาน แต่จริงๆ มี gap ตรงที่ DLS policy ไม่ enforce field นี้

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/openrag-platform|OpenRAG Platform]] — Document-level RBAC overview
