# ERD — Enhance (sau khi có SSO đồng bộ web/mobile và luồng nhúng đối tác)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng yêu cầu đề bài — liên kết session các kênh về cùng 1 `USER` để đồng bộ đăng xuất, đối tác nhúng với token phạm vi giới hạn, phát hiện brute-force/token theft, và audit log cho mọi phiên liên quan đến tiền. So với base, `SESSION` nay được nhóm theo `user_id` để Single Logout có thể huỷ đồng loạt mọi kênh, kể cả phiên nhúng ở đối tác.

```mermaid
erDiagram
    USER ||--o{ SESSION : "logs in via"
    USER ||--o{ PARTNER_EMBED_GRANT : "authorizes"
    PARTNER ||--o{ PARTNER_EMBED_GRANT : "receives"
    SESSION ||--o{ REVOCATION_EVENT : "may trigger"
    USER ||--o{ LOGIN_ATTEMPT : "makes"
    SESSION ||--o{ AUDIT_LOG : "generates"

    USER {
        string user_id PK
        string email
        string password_hash
        string kyc_status
    }

    SESSION {
        string session_id PK
        string user_id FK
        string channel
        string state
        string nonce
        datetime issued_at
        datetime expires_at
    }

    PARTNER {
        string partner_id PK
        string name
    }

    PARTNER_EMBED_GRANT {
        string grant_id PK
        string user_id FK
        string partner_id FK
        string session_id FK
        string scope
        datetime issued_at
        datetime revoked_at
    }

    REVOCATION_EVENT {
        string event_id PK
        string session_id FK
        string reason
        datetime triggered_at
    }

    LOGIN_ATTEMPT {
        string attempt_id PK
        string user_id FK
        string source_ip
        string result
        string failure_reason
        datetime attempted_at
    }

    AUDIT_LOG {
        string log_id PK
        string session_id FK
        string user_id FK
        string channel
        string device_info
        string action
        datetime created_at
    }
```
