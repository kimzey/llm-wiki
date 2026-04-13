# allowed_groups ทำงานยังไงกับ Google Drive — การวิเคราะห์เชิงลึก

> **สรุปคำตอบสั้น:** `allowed_groups` เก็บ Google Workspace group email ได้ถูกต้องตอน ingest
> แต่ **ไม่ถูก enforce ที่ query time** เพราะ DLS policy ของ OpenSearch ไม่ได้ check field นี้
> นี่คือ **architectural gap** ที่ต้องแก้ไขเพิ่มเติมถ้าต้องการ group-based access control

---

## สารบัญ

- [ภาพรวมการไหลของข้อมูล](#ภาพรวมการไหลของข้อมูล)
- [ขั้นตอน 1: Ingest — Google Drive ดึง ACL มายังไง](#ขั้นตอน-1-ingest--google-drive-ดึง-acl-มายังไง)
- [ขั้นตอน 2: Index — ข้อมูลถูกเก็บใน OpenSearch ยังไง](#ขั้นตอน-2-index--ข้อมูลถูกเก็บใน-opensearch-ยังไง)
- [ขั้นตอน 3: Query — DLS Filter ทำงานยังไง](#ขั้นตอน-3-query--dls-filter-ทำงานยังไง)
- [ปัญหา: allowed_groups ถูก ignore](#ปัญหา-allowed_groups-ถูก-ignore)
- [ทำไม JWT ไม่มี group memberships](#ทำไม-jwt-ไม่มี-group-memberships)
- [สรุปตาราง: อะไรทำงาน อะไรไม่ทำงาน](#สรุปตาราง-อะไรทำงาน-อะไรไม่ทำงาน)
- [วิธีแก้ไข (ถ้าต้องการ)](#วิธีแก้ไข-ถ้าต้องการ)

---

## ภาพรวมการไหลของข้อมูล

```
Google Drive
    │
    ├── File: "Q3 Budget.pdf"
    │       Permissions:
    │         - user: alice@company.com (reader)
    │         - group: finance-team@company.com (reader)
    │
    ▼
[Google Drive Connector]
    │   calls permissions().list() API
    │
    ▼
allowed_users = ["alice@company.com"]        ← type == "user"
allowed_groups = ["finance-team@company.com"] ← type == "group"
    │
    ▼
[OpenSearch Chunk stored]
{
  "filename": "Q3 Budget.pdf",
  "owner": "admin-user-uuid",
  "allowed_users": ["alice@company.com"],
  "allowed_groups": ["finance-team@company.com"],  ← ✅ เก็บถูกต้อง
  "text": "...content..."
}
    │
    ▼
[Bob queries: "what is Q3 budget?"]
    │
    ▼
OpenSearch DLS filter runs:
  - owner == bob-uuid?         ← ❌ ไม่ใช่
  - allowed_users has bob-uuid? ← ❌ ไม่มี
  - document has no owner?     ← ❌ มี owner
    │
    ▼
❌ Bob ไม่ได้รับผล — แม้ Bob จะเป็นสมาชิกของ finance-team@company.com
```

---

## ขั้นตอน 1: Ingest — Google Drive ดึง ACL มายังไง

**ไฟล์:** `src/connectors/google_drive/connector.py`

```python
def _extract_acl(self, file_metadata: dict) -> DocumentACL:
    permissions = file_metadata.get("permissions", [])

    allowed_users = []
    allowed_groups = []

    for permission in permissions:
        perm_type = permission.get("type", "")
        email = permission.get("emailAddress", "")

        if perm_type == "user" and email:
            allowed_users.append(email)          # เก็บ user email
        elif perm_type == "group" and email:
            allowed_groups.append(email)         # เก็บ group email
        # "domain" type ถูก ignore (Google Workspace domain-wide sharing)

    return DocumentACL(
        owner=self._get_owner_id(),
        allowed_users=allowed_users,
        allowed_groups=allowed_groups,
    )
```

**ตัวอย่างผลลัพธ์:**

| File | allowed_users | allowed_groups |
|------|---------------|----------------|
| budget.pdf | alice@co.com, bob@co.com | finance@co.com |
| hr-policy.pdf | hr-manager@co.com | hr-team@co.com, all-staff@co.com |
| public-docs.pdf | (empty) | (empty) |

---

## ขั้นตอน 2: Index — ข้อมูลถูกเก็บใน OpenSearch ยังไง

**ไฟล์:** `src/config/settings.py` (index mapping)

```python
"allowed_users": {"type": "keyword"},   # array of email strings
"allowed_groups": {"type": "keyword"},  # array of group email strings
```

ทุก chunk ของเอกสารมีฟิลด์เหล่านี้ครบ — เก็บ group email ได้ถูกต้องสมบูรณ์

---

## ขั้นตอน 3: Query — DLS Filter ทำงานยังไง

**ไฟล์:** `securityconfig/roles.yml`

```yaml
openrag_user_role:
  description: "DLS: user can read/write docs they own or are allowed on"
  index_permissions:
    - index_patterns: ["documents", "documents*", ...]
      allowed_actions:
        - crud
      dls: >
        {"bool":{"should":[
          {"term":{"owner":"${user.name}"}},
          {"term":{"allowed_users":"${user.name}"}},
          {"bool":{"must_not":{"exists":{"field":"owner"}}}}
        ],"minimum_should_match":1}}
```

**OpenSearch DLS อธิบาย:**

`${user.name}` = ค่าจาก JWT claim `sub` (user UUID เช่น `abc123-...`)

ทุกครั้งที่ user query OpenSearch จะ inject filter นี้ก่อน ผู้ใช้เห็นเฉพาะ doc ที่:
1. `owner == user.sub` → user เป็นเจ้าของ
2. `allowed_users contains user.sub` → user ถูกระบุชื่อไว้
3. `owner field ไม่มี` → เอกสารสาธารณะ

**ไม่มี condition ที่ 4:** `allowed_groups contains ???`

---

## ปัญหา: allowed_groups ถูก ignore

```
ปัญหา 1: DLS ไม่มี allowed_groups check
──────────────────────────────────────────────────────
DLS ปัจจุบัน:
  {"term":{"owner":"${user.name}"}},
  {"term":{"allowed_users":"${user.name}"}},
  {"bool":{"must_not":{"exists":{"field":"owner"}}}}

ขาด:
  {"term":{"allowed_groups":"???"}}   ← ไม่มีใน policy!


ปัญหา 2: ถ้าเพิ่ม allowed_groups check — จะ match กับค่าอะไร?
──────────────────────────────────────────────────────
allowed_groups เก็บ:  "finance-team@company.com"
${user.name} คือ:     "abc123-def456-..." (UUID จาก Google sub)

→ ไม่มีทางที่ UUID จะ match group email ได้!


ปัญหา 3: JWT ไม่มี group membership claims
──────────────────────────────────────────────────────
JWT payload ที่ OpenRAG สร้าง (session_manager.py):
{
  "sub": "google-oauth2|abc123",
  "email": "bob@company.com",
  "roles": ["openrag_user"],
  "iat": 1234567890,
  "exp": 1235000000
}

ไม่มี: "groups": ["finance-team@company.com", ...]
```

---

## ทำไม JWT ไม่มี group memberships

**ไฟล์:** `src/session_manager.py`

```python
def create_jwt(self, user_id: str, email: str, ...) -> str:
    payload = {
        "sub": user_id,         # Google user ID
        "email": email,
        "roles": ["openrag_user"],
        "iat": now,
        "exp": now + timedelta(days=7),
    }
    return jwt.encode(payload, private_key, algorithm="RS256")
```

ตอน login ผ่าน Google OAuth:
- ระบบได้ `access_token` สำหรับ Google APIs
- แต่ **ไม่ได้ call Google Directory API** เพื่อดึง group memberships
- group memberships ของ user ไม่ถูก fetch / ไม่ถูกใส่ใน JWT

---

## สรุปตาราง: อะไรทำงาน อะไรไม่ทำงาน

| Feature | สถานะ | รายละเอียด |
|---------|--------|------------|
| เก็บ group email ตอน ingest (Google Drive) | ✅ ทำงาน | `_extract_acl()` ดึงจาก permissions API ถูกต้อง |
| เก็บ `allowed_users` ตอน ingest | ✅ ทำงาน | ใส่ user email ได้ถูกต้อง |
| enforce `owner` check ที่ query time | ✅ ทำงาน | DLS `${user.name}` match กับ `owner` field |
| enforce `allowed_users` check ที่ query time | ⚠️ บางส่วน | DLS check แต่ใช้ `user.sub` (UUID) ส่วน allowed_users บางอันอาจเก็บ email |
| enforce `allowed_groups` check ที่ query time | ❌ ไม่ทำงาน | DLS policy ไม่มี condition นี้เลย |
| JWT มี group membership | ❌ ไม่มี | ไม่ได้ call Google Directory API |

> **หมายเหตุ `allowed_users`:** connector เก็บ email (`alice@company.com`) แต่ DLS ใช้ `${user.name}` ซึ่งเป็น `sub` (UUID) — อาจมี mismatch ด้วยเช่นกัน ขึ้นกับว่า owner UUID ตรงกับที่เก็บไว้หรือไม่

---

## วิธีแก้ไข (ถ้าต้องการ)

### Option A: ขยาย JWT ให้มี group claims

**1. Fetch groups ตอน Google OAuth login:**
```python
# src/api/auth.py — ตอนรับ callback
async def google_callback(request: Request):
    token = await oauth.google.authorize_access_token(request)

    # เดิม: แค่ดึง userinfo
    userinfo = token["userinfo"]

    # เพิ่ม: call Google Directory API
    credentials = Credentials(token["access_token"])
    service = build("admin", "directory_v1", credentials=credentials)
    groups_result = service.groups().list(
        userKey=userinfo["email"]
    ).execute()

    group_emails = [g["email"] for g in groups_result.get("groups", [])]
    # → ["finance-team@company.com", "all-staff@company.com"]
```

> ⚠️ ต้องใช้ scope `https://www.googleapis.com/auth/admin.directory.group.readonly`
> และ service account ที่มี domain-wide delegation หรือ admin consent

**2. ใส่ groups ใน JWT:**
```python
def create_jwt(self, user_id, email, groups=None):
    payload = {
        "sub": user_id,
        "email": email,
        "roles": ["openrag_user"],
        "groups": groups or [],   # ← เพิ่ม
    }
```

**3. อัพเดท OpenSearch DLS policy:**
```yaml
dls: >
  {"bool":{"should":[
    {"term":{"owner":"${user.name}"}},
    {"term":{"allowed_users":"${user.name}"}},
    {"terms":{"allowed_groups":${user.roles}}},
    {"bool":{"must_not":{"exists":{"field":"owner"}}}}
  ],"minimum_should_match":1}}
```

> `${user.roles}` ใน OpenSearch DLS interpolation รองรับ array —
> ถ้า JWT มี `"groups": [...]` ต้องใช้ custom claim หรือ roles substitution

---

### Option B: ใช้ Webhook ACL Sync แทน

แทนที่จะพึ่ง JWT, ทำ real-time sync:
- Google Drive webhook notify เมื่อ permissions เปลี่ยน
- Backend call `batch_update_acls()` (ใน `acl_utils.py` — มีอยู่แล้ว!)
- แปลง group email → user emails โดย lookup Google Directory API ฝั่ง backend
- เก็บ user emails ลงใน `allowed_users` แทน group emails

**ข้อดี:** ไม่ต้องแก้ DLS, ไม่ต้องแก้ JWT
**ข้อเสีย:** ต้อง sync ทุกครั้ง group membership เปลี่ยน, ข้อมูลซ้ำซ้อน

---

### Option C: ไม่ต้องแก้อะไร — ใช้แค่ allowed_users

ถ้า Google Drive ตั้งค่า permission ระดับ user ไม่ใช่ group:
- Connector จะเก็บ email ลง `allowed_users` ได้ถูกต้อง
- `allowed_groups` จะ empty
- ระบบทำงานได้ปกติ

**เหมาะกับองค์กรที่:**
- Share ไฟล์แบบ individual user ไม่ใช่ group
- ไม่มี Google Workspace Groups
- ต้องการ permission ที่ละเอียดระดับ user

---

## สรุปสุดท้าย

```
allowed_groups กับ Google ทำงานได้แค่ครึ่งเดียว:

✅ ตอน INGEST:
   Google Drive Connector → permissions().list() → type=="group" → เก็บ group email
   ข้อมูลถูกต้องสมบูรณ์ใน OpenSearch

❌ ตอน QUERY:
   OpenSearch DLS → ตรวจแค่ owner และ allowed_users
   allowed_groups field ถูก ignore ทั้งหมด
   ผู้ใช้ที่เป็นสมาชิก group ไม่ได้รับสิทธิ์

Gap นี้ต้องแก้ไขเพิ่มเติมถ้าต้องการ group-based access control:
→ Option A (แนะนำ): เพิ่ม groups claim ใน JWT + อัพเดท DLS policy
→ Option B: Expand groups → users ฝั่ง backend ตอน sync
→ Option C: หลีกเลี่ยงการใช้ Google Groups, share ระดับ user แทน
```
