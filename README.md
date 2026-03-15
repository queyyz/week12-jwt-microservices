# 🔐 Task Board — Security Architecture (Term Project)

## 📋 รายละเอียดใบงาน

ใบงานนี้เป็นส่วนหนึ่งของ Term Project วิชา ENGSE207 Software Architecture โดยมีวัตถุประสงค์เพื่อ:

1. **ทดสอบ Security** — ทำ Test Cases 8 ข้อ เพื่อตรวจสอบระบบ Authentication (JWT), Authorization (RBAC), Rate Limiting, และ SQL Injection Prevention
2. **เปรียบเทียบ Architecture** — วาด C2 Diagram เปรียบเทียบสถาปัตยกรรมก่อน (Week 6 — Monolithic) และหลัง (Week 12 — Microservices + Security) พร้อมอธิบาย Components ที่เพิ่มมาและ STRIDE Threats ที่แก้ไข
3. **เขียน ADR** — Architecture Decision Record สำหรับการตัดสินใจ "เพิ่ม Auth Service แยกจาก Task Service"

## 📁 โครงสร้างโปรเจค

```
task-board-security/
├── auth-service/          # 🔑 Authentication Service (JWT, bcrypt, login/register)
├── task-service/          # 📋 Task CRUD Service (protected by JWT + RBAC)
├── user-service/          # 👤 User Profile Service (protected by JWT + RBAC)
├── nginx/                 # 🛡️ API Gateway (Rate Limiting, Security Headers)
├── frontend/              # 🌐 Frontend (Static HTML/CSS/JS)
├── monitoring/            # 📊 Monitoring Stack (Grafana, Loki, Promtail)
├── docs/
│   └── ADR-001-auth-service.md   # Architecture Decision Record
├── docker-compose.yml     # Docker Compose orchestration
├── REPORT.md              # ผลการทดสอบ + Architecture Comparison
└── README.md              # ไฟล์นี้
```

## 🛡️ Security Features

| Feature | Component | รายละเอียด |
|---------|-----------|------------|
| JWT Authentication | Auth Service | ออก token พร้อม bcrypt password hashing |
| Rate Limiting | Nginx Gateway | auth: 5r/m, api: 30r/m ต่อ IP |
| Security Headers | Nginx Gateway | X-Frame-Options, X-Content-Type-Options, XSS Protection |
| RBAC | Task/User Service | requireAuth + requireRole middleware |
| SQL Injection Prevention | ทุก Service | Parameterized queries ($1, $2, ...) |
| Timing Attack Prevention | Auth Service | bcrypt.compare กับ dummy hash |
| Network Isolation | Docker | Private bridge network (taskboard-net) |
| Centralized Logging | Monitoring Stack | Promtail → Loki → Grafana |

## 🚀 วิธีรัน

```bash
# Start ทุก services
docker-compose up -d

# เข้าใช้งาน
# Frontend:  http://localhost
# Grafana:   http://localhost:3030 (admin/admin)
```

## 📄 เอกสารที่เกี่ยวข้อง

- [REPORT.md](./REPORT.md) — ผลทดสอบ 8 Test Cases + Architecture Comparison
- [ADR-001-auth-service.md](./docs/ADR-001-auth-service.md) — Architecture Decision Record

---

## 👤 ผู้จัดทำ
**ชื่อ:** ธนพล ตรีรัตนานุภาพ 
**รหัสนักศึกษา:** 67543210070-8  
**รายวิชา:** ENGSE207 Software Architecture  
**สัปดาห์:** Week 7  

---
