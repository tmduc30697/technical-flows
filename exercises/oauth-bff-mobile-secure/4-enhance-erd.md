# Enhance ERD — Thêm liên kết Open Banking qua BFF với PKCE và audit trail

Đây là ERD **sau khi** enhance được áp dụng lên base. Các entity gốc (User, Device, Session, BankAccount) giữ nguyên. So với base, có 3 nhóm entity mới, ứng trực tiếp với các yêu cầu trong đề bài:

- `OAUTH_LINK_REQUEST` (mới) — theo dõi 1 phiên liên kết đang diễn ra: `state` và `code_challenge` (PKCE) được BFF lưu tạm để đối chiếu khi partner bank redirect trở lại, đồng thời map đúng về `device_id` đã khởi tạo để xử lý deep link/universal link.
- `LINKED_BANK_CONNECTION` (mới) — lưu access_token/refresh_token của từng ngân hàng đã liên kết **chỉ ở phía BFF**, mobile app không bao giờ thấy các token này.
- `AUDIT_LOG` (mới) — ghi nhận ai liên kết ngân hàng nào, khi nào, từ thiết bị nào, đáp ứng yêu cầu compliance Open Banking.

```mermaid
erDiagram
    USER ||--o{ DEVICE : "sở hữu"
    USER ||--o{ SESSION : "đăng nhập"
    DEVICE ||--o{ SESSION : "khởi tạo từ"
    USER ||--o{ BANK_ACCOUNT : "sở hữu tài khoản chính"
    USER ||--o{ LINKED_BANK_CONNECTION : "liên kết"
    USER ||--o{ OAUTH_LINK_REQUEST : "khởi tạo"
    DEVICE ||--o{ OAUTH_LINK_REQUEST : "từ thiết bị"
    OAUTH_LINK_REQUEST ||--o| LINKED_BANK_CONNECTION : "hoàn tất thành"
    USER ||--o{ AUDIT_LOG : "có hành động"
    DEVICE ||--o{ AUDIT_LOG : "từ thiết bị"

    USER {
        string id PK
        string full_name
        string phone
        string kyc_status
    }
    DEVICE {
        string id PK
        string user_id FK
        string device_type
        string push_token
        datetime registered_at
    }
    SESSION {
        string id PK
        string user_id FK
        string device_id FK
        string session_token "token nội bộ của app"
        datetime created_at
        datetime expires_at
    }
    BANK_ACCOUNT {
        string id PK
        string user_id FK
        string account_number
        decimal balance
        string currency
    }
    OAUTH_LINK_REQUEST {
        string id PK
        string user_id FK
        string device_id FK
        string partner_bank_id
        string state
        string code_challenge "PKCE, code_verifier chỉ giữ ở mobile app"
        string status "pending | completed | failed"
        datetime created_at
        datetime expires_at
    }
    LINKED_BANK_CONNECTION {
        string id PK
        string user_id FK
        string partner_bank_id
        string external_account_ref
        string access_token_encrypted "chỉ BFF giữ, mobile không thấy"
        string refresh_token_encrypted "chỉ BFF giữ, mobile không thấy"
        datetime token_expires_at
        string status "active | revoked"
        datetime linked_at
    }
    AUDIT_LOG {
        string id PK
        string user_id FK
        string device_id FK
        string partner_bank_id
        string event_type "link_started | link_completed | token_refreshed"
        datetime created_at
        string ip_address
    }
```
