# ERD — Base (trước khi có cổng đăng nhập chung SSO nội bộ)

Đây là **base**: mô hình dữ liệu suy luận cho các app nội bộ doanh nghiệp *trước khi* có cổng đăng nhập chung. Đề bài nói rõ "mỗi app hiện có login riêng" — nghĩa là base đã có nhân viên (từ hệ thống HR) và mỗi app tự quản lý tài khoản/session cục bộ riêng biệt, chưa có IdP trung tâm, chưa chia sẻ session, chưa có Single Logout.

```mermaid
erDiagram
    EMPLOYEE ||--o{ APP_ACCOUNT : "has account in"
    APP ||--o{ APP_ACCOUNT : "manages"
    APP_ACCOUNT ||--o{ APP_SESSION : "logs into"

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

    APP_ACCOUNT {
        string account_id PK
        string employee_id FK
        string app_id FK
        string password_hash
        string status
    }

    APP_SESSION {
        string session_id PK
        string account_id FK
        datetime issued_at
        datetime expires_at
    }
```
