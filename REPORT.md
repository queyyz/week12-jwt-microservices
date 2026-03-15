# 📝 Security Testing & Architecture Report
## Task Board — Week 12 Security Architecture

---

## ส่วนที่ 1: สรุปผลการทดสอบ (Test Results)

กรอกตารางนี้หลังทำ Test Cases ครบ:

| # | Test Case | Expected | Actual | ✅/❌ | หมายเหตุ |
|---|-----------|----------|--------|-------|----------|
| 1 | ไม่มี Token | 401 | 401 | ✅ | `GET /api/tasks/` โดยไม่ส่ง Authorization header → middleware `requireAuth` ตรวจพบว่าไม่มี token แล้ว return `401 Unauthorized` พร้อมข้อความ "กรุณา Login ก่อน" |
| 2 | Login สำเร็จ | 200 + Token | 200 + Token | ✅ | `POST /api/auth/login` ด้วย `alice@example.com` / `password123` → ได้ status 200 พร้อม JWT token และข้อมูล user กลับมา |
| 3 | มี Token ถูกต้อง | 200 + data | 200 + data | ✅ | `GET /api/tasks/` พร้อม `Authorization: Bearer <valid_token>` → middleware verify สำเร็จ ได้รับ tasks data กลับมา |
| 4 | Token Invalid | 401 | 401 | ✅ | `GET /api/tasks/` พร้อม `Authorization: Bearer invalid.token.here` → middleware `jwt.verify()` throw error → return `401 Invalid Token` |
| 5 | Forbidden (403) | 403 | 403 | ✅ | `PUT /api/tasks/:id` โดย member ที่ไม่ใช่เจ้าของ task → route handler ตรวจ `owner_id !== req.user.sub` และ `role !== 'admin'` → return `403 Forbidden` |
| 6 | Admin access | 200 | 200 | ✅ | `GET /api/users/` ด้วย token ที่มี `role: 'admin'` → middleware `requireRole('admin')` ผ่าน → ได้รับรายชื่อ users ทั้งหมด status 200 |
| 7 | Rate Limit | 429 | 429 | ✅ | ส่ง request ไปที่ `/api/auth/login` มากกว่า 5 ครั้งต่อนาที → Nginx `limit_req zone=auth_limit` ทำงาน → return `429 Too Many Requests` |
| 8 | SQL Injection | Safe | Safe | ✅ | ส่ง `email: "' OR 1=1 --"` ใน login → ใช้ parameterized query (`$1`) ป้องกัน injection ได้ → ไม่มี data leak, ได้ 401 ปกติ |

### สรุป
- ผ่านครบ **8/8** test cases
- ระบบมีการป้องกันเรื่อง Authentication (JWT), Authorization (RBAC), Rate Limiting (Nginx), และ SQL Injection (Parameterized Queries) อย่างครบถ้วน

---

## ส่วนที่ 2: Architecture Comparison (C2 Diagram)

### 🔵 ก่อน (Week 6) — Monolithic Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      Client (Browser)                   │
└──────────────────────────┬──────────────────────────────┘
                           │ HTTP
                           ▼
┌─────────────────────────────────────────────────────────┐
│                   Task Service (Node.js)                │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌────────────────┐  │
│  │ Auth Logic   │  │ Task CRUD   │  │ User Profile   │  │
│  │ (login/reg)  │  │ (REST API)  │  │ Management     │  │
│  └─────────────┘  └─────────────┘  └────────────────┘  │
│                                                         │
│  ❌ ไม่มี Rate Limiting                                  │
│  ❌ ไม่มี Security Headers                               │
│  ❌ ไม่มี Logging/Monitoring                              │
│  ❌ Auth อยู่รวมกับ Business Logic                        │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
              ┌────────────────────────┐
              │   PostgreSQL Database  │
              │   (Single Database)    │
              └────────────────────────┘
```

### 🟢 หลัง (Week 12) — Microservices + Security Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Client (Browser)                                  │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ HTTP (port 80)
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    🛡️ Nginx API Gateway (NEW)                               │
│                                                                             │
│  ✅ Rate Limiting (auth_limit: 5r/m, api_limit: 30r/m)                     │
│  ✅ Security Headers (X-Frame-Options, X-Content-Type-Options, XSS, etc.)  │
│  ✅ Reverse Proxy + JWT Header Forwarding                                   │
│  ✅ Static File Serving (Frontend)                                          │
└───────┬────────────────────────┬────────────────────────┬───────────────────┘
        │                        │                        │
        ▼                        ▼                        ▼
┌───────────────┐  ┌──────────────────┐  ┌────────────────────┐
│ 🔑 Auth       │  │ 📋 Task          │  │ 👤 User            │
│ Service (NEW) │  │ Service          │  │ Service (NEW)      │
│ Port: 3001    │  │ Port: 3002       │  │ Port: 3003         │
│               │  │                  │  │                    │
│ • Register    │  │ • CRUD Tasks     │  │ • Profile /me      │
│ • Login       │  │ • requireAuth    │  │ • requireAuth      │
│ • JWT Issue   │  │ • requireRole    │  │ • requireRole      │
│ • Verify      │  │ • Owner Check    │  │ • Admin-only list  │
│ • bcrypt Hash │  │                  │  │                    │
│ • Timing Atk  │  │                  │  │                    │
│   Prevention  │  │                  │  │                    │
└───────┬───────┘  └────────┬─────────┘  └─────────┬──────────┘
        │                   │                      │
        ▼                   ▼                      ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
│   auth_db    │  │   task_db    │  │    user_db       │
│ (PostgreSQL) │  │ (PostgreSQL) │  │  (PostgreSQL)    │
└──────────────┘  └──────────────┘  └──────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    📊 Monitoring Stack (NEW)                                │
│                                                                             │
│  ┌──────────┐     ┌──────────┐     ┌──────────────┐                        │
│  │ Promtail │────▶│   Loki   │────▶│   Grafana    │                        │
│  │ (Log     │     │ (Log     │     │ (Dashboard   │                        │
│  │  Agent)  │     │  Store)  │     │  & Alerts)   │                        │
│  └──────────┘     └──────────┘     └──────────────┘                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Components ที่เพิ่มมา และเหตุผล (STRIDE Mapping)

| Component ที่เพิ่ม | STRIDE Threat ที่แก้ | อธิบาย |
|---|---|---|
| **Nginx API Gateway** | **Denial of Service (DoS)** | เพิ่ม Rate Limiting (`auth_limit: 5r/m`, `api_limit: 30r/m`) เพื่อป้องกัน brute-force และ API abuse |
| **Nginx Security Headers** | **Tampering / Information Disclosure** | เพิ่ม `X-Frame-Options: DENY` (กัน Clickjacking), `X-Content-Type-Options: nosniff` (กัน MIME sniffing), `X-XSS-Protection` (กัน XSS), `Referrer-Policy: no-referrer` (กัน information leak) |
| **Auth Service (แยกจาก Task)** | **Spoofing / Elevation of Privilege** | แยก Authentication logic ออกเป็น service แยก เพื่อใช้ JWT + bcrypt อย่างเหมาะสม ป้องกัน identity spoofing และมี Timing Attack prevention |
| **User Service (แยก)** | **Elevation of Privilege** | แยก RBAC (Role-Based Access Control) ออกมา — admin เท่านั้นที่ดู user list ได้ ป้องกัน privilege escalation |
| **Database per Service** | **Information Disclosure / Tampering** | แยก DB ออก 3 ตัว (auth_db, task_db, user_db) ป้องกัน blast radius เมื่อ service ใด service หนึ่งถูก compromise |
| **Monitoring Stack** (Grafana, Loki, Promtail) | **Repudiation** | เก็บ log ทุก request (morgan → Docker logs → Promtail → Loki → Grafana) ป้องกัน non-repudiation ทำให้ track ได้ว่าใครทำอะไร |
| **Docker Network Isolation** | **Information Disclosure** | Services ทั้งหมดอยู่ใน `taskboard-net` (bridge network) ภายใน Docker — ไม่ expose port ของ services ออกสู่ภายนอก เฉพาะ Nginx port 80 เท่านั้น |
| **Parameterized Queries** | **Tampering (SQL Injection)** | ใช้ parameterized queries (`$1`, `$2`, ...) ในทุก SQL query ป้องกัน SQL Injection ทั้งระบบ |

### สรุปการเปลี่ยนแปลง

จาก **Week 6** ที่เป็น Monolithic (1 Service, 1 Database, ไม่มี Security Layer) เปลี่ยนเป็น **Week 12** ที่เป็น Microservices Architecture พร้อม Security Controls ครบถ้วนตาม STRIDE Model ทั้ง 6 ด้าน:

1. **Spoofing** → JWT Authentication + bcrypt + Timing Attack Prevention
2. **Tampering** → Parameterized Queries + Security Headers
3. **Repudiation** → Centralized Logging (Promtail → Loki → Grafana)
4. **Information Disclosure** → Database Isolation + Network Isolation + Security Headers
5. **Denial of Service** → Nginx Rate Limiting
6. **Elevation of Privilege** → RBAC (requireRole middleware) + Owner-based Authorization
