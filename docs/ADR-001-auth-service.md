# ADR-001: เพิ่ม Auth Service แยกจาก Task Service

## Status
Accepted

## Context
ในสถาปัตยกรรมเดิม (Week 6) ระบบถูกออกแบบมาให้ Task Service (หรือ Backend หลัก) ทำหน้าที่รวมศูนย์ทุกอย่าง ทั้งการจัดการข้อมูล Task, การจัดการข้อมูล User และการทำ Authentication (Login/Register) ไว้ในที่เดียวกัน (Monolithic structure ในส่วนของ API) 
การผูก Logic การพิสูจน์ตัวตนไว้กับ Service ที่จัดการเอกสาร/งาน ส่งผลให้เกิดความซับซ้อนของ Codebase และหากมีปริมาณคนพยายาม Login หรือ Register เข้ามาพร้อมกันจำนวนมาก อาจดึงทรัพยากรระบบไปจนทำให้ผู้ใช้ที่ต้องการดึงข้อมูล Task ธรรมดาๆ ได้รับผลกระทบ (Performance limitation) นอกจากนี้ ในแง่ของความปลอดภัย หาก Backend ตัวเดียวนี้โดนเจาะ แฮกเกอร์อาจเข้าถึงได้ทั้งโค้ดจัดการ Token และข้อมูลทั้งหมดทันที

## Decision
ตัดสินใจแยก Auth Service ออกเป็น Service แยกจาก Task Service และ User Service อย่างเด็ดขาด โดย Auth Service จะมีหน้าที่หลักคือ:
- รับผิดชอบกระบวนการยืนยันตัวตน (Authentication) เท่านั้น (Login, Register)
- เชื่อมต่อกับ User Database เพื่อนำ Email และ Password มาตรวจสอบ (Hash Compare)
- ออกใบรับรองสิทธิ์รูปแบบ JSON Web Token (JWT) เพื่อให้ Client เก็บและนำไปใช้
- Service อื่นๆ (เช่น Task Service) จะเปลี่ยนหน้าที่ไปกระทำเพียงแค่ "การตรวจสอบความถูกต้องของ JWT (Verify)" และบังคับใช้สิทธิ์ (Authorization) จาก Payload ที่อ่านได้ โดยไม่ต้องเข้ามาจัดการกระบวนการ Login อีกต่อไป

## Consequences
**Positive:**
- **Separation of Concerns:** หน้าที่การทำงานแต่ละระบบถูกแบ่งแยกชัดเจน โค้ดของ Task Service จะเบาลงและโฟกัสแค่ทางด้าน Business Logic ของงานเท่านั้น
- **Independent Scalability:** สามารถ Scale เพิ่ม/ลดทรัพยากรของ Auth Service แยกกันกับ Task Service ได้อย่างอิสระ ตามปริมาณผู้ใช้งานที่กำลัง Login โหลดหนักแค่ Auth แต่ Task ดึงข้อมูลได้ราบรื่น
- **Enhanced Security:** รวมศูนย์การควบคุมเกร็ดความปลอดภัย การเข้ารหัส Token และ Secret Key ไว้ที่ Service เดียว จำกัดขอบเขตช่องโหว่ (Attack Surface) ได้ดีขึ้น

**Negative:**
- **Network & Operational Overhead:** มีความซับซ้อนทางด้าน Architecture มากขึ้น ต้องบริหารจัดการ Microservice ที่เพิ่มขึ้น, การจัดการ Container และการเซ็ตอัพ API Gateway ที่ต้อง Forward Request ให้ตรงจุด
- **Complexity in Data Sharing:** ข้อมูล User Profile บางส่วนที่ต้องแสดงผลอาจไม่ครบถ้วนใน Auth หรือไม่ได้ถือครองไว้ ต้องออกแบบว่าจะส่งต่อ (Share) Data ระหว่าง Service อย่างไร

**Trade-offs:**
- ยอมแลกความซับซ้อนในระดับ Infrastructure (ต้องดูแล Service เพิ่ม) และ Latency เล็กน้อยในการสื่อสารผ่านเครือข่าย เพื่อแลกกับ Security ที่รัดกุมขึ้น, อัตรา Availability ของระบบโดยรวมที่ดีขึ้น (ไม่ล่มแบบ Single Point of Failure ง่ายๆ) และสถาปัตยกรรมที่พร้อมสเกลตัวได้ในอนาคต