# ERD — Enhance (sau khi có phân quyền per-app, đồng bộ bảo mật, revoke theo phạm vi)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng yêu cầu đề bài — quyền riêng từng app tách khỏi token trung tâm, sự kiện bảo mật lan truyền để app yêu cầu re-auth, ngữ cảnh cá nhân/tổ chức, cache/fallback khi IdP quá tải, và revoke theo phạm vi từng app. So với base, `APP_SESSION` không còn tự mang toàn quyền mà mỗi app phải tự tra `APP_PERMISSION` sau khi xác thực.

```mermaid
erDiagram
    USER ||--o{ APP_SESSION : "logs into apps via"
    APP ||--o{ APP_SESSION : "trusted by"
    USER ||--o{ APP_PERMISSION : "granted per app"
    APP ||--o{ APP_PERMISSION : "defines"
    USER ||--o{ ACCOUNT_CONTEXT : "has"
    USER ||--o{ SECURITY_EVENT : "triggers"
    APP_SESSION ||--o| APP_GRANT : "scoped by"

    USER {
        string user_id PK
        string email
        string password_hash
        datetime created_at
    }

    ACCOUNT_CONTEXT {
        string context_id PK
        string user_id FK
        string type
        string org_id
    }

    APP {
        string app_id PK
        string name
    }

    APP_PERMISSION {
        string permission_id PK
        string user_id FK
        string app_id FK
        string role
        string context_id FK
    }

    APP_SESSION {
        string session_id PK
        string user_id FK
        string app_id FK
        string context_id FK
        string central_token
        datetime issued_at
        datetime expires_at
        string reauth_required
    }

    APP_GRANT {
        string grant_id PK
        string session_id FK
        string scope
        datetime revoked_at
    }

    SECURITY_EVENT {
        string event_id PK
        string user_id FK
        string event_type
        datetime triggered_at
        boolean propagated
    }
```
