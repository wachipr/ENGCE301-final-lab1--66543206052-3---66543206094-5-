# รายชื่อสมาชิก
**66543206052-3 นางสาววชิรานีย์ ประเสริฐศรี**
**66543206094-5 นายอานุภาพ เฑียรประยูร**

## Architecture diagram
```mermaid
graph TB
    subgraph "External"
        Client[Client Browser] -->|HTTPS 443| Nginx
        Client -->|HTTP 80| Redirect[HTTP → HTTPS Redirect]
    end

    subgraph "DMZ / Reverse Proxy"
        Nginx[Nginx API Gateway] -->|Rate Limit| Auth[Auth Service:3001]
        Nginx -->|Rate Limit| Task[Task Service:3002]
        Nginx -->|Rate Limit| Log[Log Service:3003]
        Nginx -->|Static Files| Frontend[Frontend:80]
        
        Nginx -->|Internal Log| Log
    end

    subgraph "Internal Services"
        Auth -->|JWT Verify| Task
        Auth -->|JWT Verify| Log
        Task -->|DB Query| PG[(PostgreSQL)]
        Log  -->|DB Query| PG
        Auth -->|DB Query| PG
        Frontend -->|API Calls| Nginx
    end

    subgraph "Monitoring"
        Dashboard[Log Dashboard] -->|Read Logs| Log
        Dashboard -->|JWT Required| Nginx
    end

    classDef secure fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    classDef internal fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    classDef external fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    
    class Client,Redirect external
    class Nginx,Auth,Task,Log,Frontend secure
    class PG,Dashboard internal
```
## การติดตั้งและใช้งาน

### ขั้นตอนการติดตั้ง

1. **ดาวน์โหลดโปรเจค**
   ```bash
   git clone <repository-url>
   cd finallab
   ```

2. **ตั้งค่า Environment Variables**
   ```bash
   cp .env.example .env
   ```
   
   แก้ไขไฟล์ `.env` ตามต้องการ (สามารถใช้ค่า default ได้)

3. **สร้าง SSL Certificate สำหรับ development**
   ```bash
   ./scripts/gen-certs.sh
   ```
   
   คำสั่งนี้จะสร้าง self-signed certificate ใน `nginx/certs/`

4. **รันด้วย Docker Compose**
   ```bash
   docker-compose up -d
   ```

5. **รอสักครู่ให้บริการทั้งหมดขึ้น**
   ```bash
   # ตรวจสอบสถานะ
   docker-compose ps
   
   # ดู log ถ้าต้องการ
   docker-compose logs -f
   ```

6. **เข้าใช้งาน**
   - เปิดเบราว์เซอร์ไปที่: `https://localhost`
   - ระบบจะ redirect อัตโนมัติจาก HTTP ไป HTTPS

### ข้อมูลเข้าสู่ระบบเริ่มต้น

| ผู้ใช้ | อีเมล | รหัสผ่าน | Role |
|--------|--------|----------|------|
| Alice | alice@example.com | password123 | member |
| Admin | admin@example.com | password123 | admin |

## 🔒 HTTPS และ Security Flow

### TLS Termination ที่ Nginx

```
Client HTTPS Request
        ↓
    Nginx (Port 443)
        ↓ (TLS terminated here)
    Internal HTTP to Services
```

- **TLS termination** เกิดที่ Nginx ทำให้ services ภายในสามารถใช้ HTTP ได้
- **Certificate**: Self-signed สำหรับ development (สามารถเปลี่ยนเป็น certificate จริงได้)
- **Protocols**: TLS 1.2 และ 1.3 เท่านั้น

### HTTP → HTTPS Redirect

```nginx
server {
    listen 80;
    server_name localhost;
    return 301 https://$host$request_uri;
}
```

- ทุกการเข้าถึง HTTP (port 80) จะถูก redirect ไป HTTPS (port 443) อัตโนมัติ

## 🔧 การพัฒนาและ Debug

### ดู log ของแต่ละ service
```bash
# ดู log ทั้งหมด
docker-compose logs -f

# ดู log ของ service เดียว
docker-compose logs -f auth-service
docker-compose logs -f task-service
docker-compose logs -f log-service
```

### ทดสอบ API ด้วย curl

**1. ขอ token ก่อน:**
```bash
TOKEN=$(curl -s -X POST https://localhost/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"password123"}' | jq -r .token)
```

**2. ใช้ token กับ API:**
```bash
curl -H "Authorization: Bearer $TOKEN" https://localhost/api/tasks/
```