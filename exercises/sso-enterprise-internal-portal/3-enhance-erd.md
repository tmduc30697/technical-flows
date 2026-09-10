# ERD — Enhance (sau khi có cổng đăng nhập chung SSO nội bộ)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng yêu cầu đề bài — IdP trung tâm, session SSO chia sẻ giữa các subdomain/app kèm chống session fixation, Single Logout, role/group đồng bộ từ HR, và audit log tập trung. So với base, `APP_SESSION` không còn độc lập mà mọi app đều tham chiếu về 1 `SSO_SESSION` trung tâm duy nhất.

```mermaid
erDiagram
    EMPLOYEE ||--o{ SSO_SESSION : "logs in via"
    EMPLOYEE ||--o{ EMPLOYEE_ROLE : "granted"
    APP ||--o{ EMPLOYEE_ROLE : "scoped to"
    SSO_SESSION ||--o{ APP_SESSION : "propagates to"
    APP ||--o{ APP_SESSION : "trusts"
    SSO_SESSION ||--o{ AUDIT_LOG : "generates"

    EMPLOYEE {
        string employee_id PK
        string full_name
        string email
        string hr_status
    }

    APP {
        string app_id PK
        string name
        string domain
    }

    EMPLOYEE_ROLE {
        string role_id PK
        string employee_id FK
        string app_id FK
        string role
        string source
    }

    SSO_SESSION {
        string session_id PK
        string employee_id FK
        string session_token
        datetime issued_at
        datetime expires_at
        datetime revoked_at
    }

    APP_SESSION {
        string app_session_id PK
        string session_id FK
        string app_id FK
        datetime issued_at
    }

    AUDIT_LOG {
        string log_id PK
        string session_id FK
        string employee_id FK
        string app_id FK
        string action
        datetime created_at
    }
```
