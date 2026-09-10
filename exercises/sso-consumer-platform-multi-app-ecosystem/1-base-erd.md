# ERD — Base (trước khi có phân quyền per-app, đồng bộ bảo mật, revoke theo phạm vi)

Đây là **base**: mô hình dữ liệu suy luận cho hệ sinh thái nhiều app tiêu dùng *trước khi* có các yêu cầu nâng cao. Đề bài mô tả nền tảng "đã tự làm Identity Provider trung tâm" — nghĩa là base đã có sẵn 1 danh tính người dùng dùng chung và các app con tin tưởng token trung tâm, nhưng chưa phân biệt quyền riêng từng app, chưa đồng bộ tín hiệu bảo mật, chưa tách ngữ cảnh cá nhân/tổ chức, chưa revoke theo phạm vi từng app.

```mermaid
erDiagram
    USER ||--o{ APP_SESSION : "logs into apps via"
    APP ||--o{ APP_SESSION : "trusted by"

    USER {
        string user_id PK
        string email
        string password_hash
        datetime created_at
    }

    APP {
        string app_id PK
        string name
    }

    APP_SESSION {
        string session_id PK
        string user_id FK
        string app_id FK
        string central_token
        datetime issued_at
        datetime expires_at
    }
```
