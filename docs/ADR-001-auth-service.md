# ADR-001: เพิ่ม Auth Service แยกจาก Task Service

## Status
Accepted

## Context
In the previous architecture (Week 6), the system was designed so that the Task Service (the main backend) handled multiple responsibilities in a single place. It managed Task data, User data, and also performed Authentication processes such as Login and Register. This resulted in a monolithic API structure, where several unrelated responsibilities were tightly coupled together.
Keeping authentication logic within the same service that manages task-related operations increases the complexity of the codebase. Additionally, if a large number of users attempt to log in or register at the same time, the authentication workload may consume system resources and negatively impact users who simply want to retrieve or manage tasks.
From a security perspective, placing all critical logic in one backend also creates higher risk. If the service is compromised, an attacker could potentially gain access to both authentication mechanisms (such as token generation) and application data simultaneously.

## Decision
The architecture will be updated by separating the Auth Service from the Task Service and User Service. The Auth Service will operate as an independent component dedicated solely to authentication responsibilities.

**The Auth Service will:**
- Handle user authentication processes, including Login and Register.
- Communicate with the User Database to validate credentials by comparing stored password hashes.
- Generate and issue JSON Web Tokens (JWT) for authenticated users.
- Allow client applications to store and include these tokens in future requests.
**Other services (such as the Task Service) will no longer manage login operations. Instead, they will only:**
- Verify the validity of JWT tokens received in requests.
- Authorize user access based on the information contained in the token payload.

## Consequences
**Positive:**
- **Clear Separation of Responsibilities:** Each service focuses on its own purpose. The Task Service becomes simpler and concentrates only on task-related business logic.
- **Independent Scalability:** The Auth Service and Task Service can be scaled separately depending on workload. For example, heavy login traffic affects only the Auth Service without slowing down task operations.
- **Enhanced Security:** รวมศูนย์การควบคุมเกร็ดความปลอดภัย การเข้ารหัส Token และ Secret Key ไว้ที่ Service เดียว จำกัดขอบเขตช่องโหว่ (Attack Surface) ได้ดีขึ้น

**Negative:**
- **Additional Network and Operational Complexity:** The architecture becomes more complex due to the introduction of extra microservices. This requires additional management of containers, service communication, and potentially an API Gateway to route requests correctly.
- **Data Coordination Challenges:** Some user profile information required by other services might not be directly available. Mechanisms for sharing or retrieving user data between services must be designed carefully.
**Trade-offs:**
This design accepts a slight increase in infrastructure complexity and network communication latency in exchange for stronger security, improved system availability, and a more scalable architecture that is better suited for future growth.
- ยอมแลกความซับซ้อนในระดับ Infrastructure (ต้องดูแล Service เพิ่ม) และ Latency เล็กน้อยในการสื่อสารผ่านเครือข่าย เพื่อแลกกับ Security ที่รัดกุมขึ้น, อัตรา Availability ของระบบโดยรวมที่ดีขึ้น (ไม่ล่มแบบ Single Point of Failure ง่ายๆ) และสถาปัตยกรรมที่พร้อมสเกลตัวได้ในอนาคต
