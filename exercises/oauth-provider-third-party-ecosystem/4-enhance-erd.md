# Enhance ERD — Hệ thống đóng vai trò OAuth Authorization Server cho app thứ ba

Đây là ERD **sau khi** enhance được áp dụng lên base. `USER`, `SHOP`, `SESSION`, `ORDER`, `CUSTOMER`, `INVENTORY_ITEM` giữ nguyên. So với base, có 4 nhóm entity mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `OAUTH_APP` (mới) — developer đăng ký qua Developer Portal, nhận `client_id`/`client_secret`, khai báo `redirect_uris` whitelist.
- `AUTHORIZATION_CODE` (mới) — code tạm phát ra sau khi user approve ở consent screen, chỉ dùng 1 lần (`used`), hết hạn 60 giây, mang theo `code_challenge` (PKCE) để validate ở bước đổi token.
- `APP_GRANT` (mới) — bản ghi user đã approve app nào, cho shop nào, với scope gì — là gốc để cascade revoke toàn bộ token liên quan khi user vào "App đã kết nối" để thu hồi quyền.
- `ACCESS_TOKEN` + `REFRESH_TOKEN` (mới) — access token mang theo `scopes` đã approve để resource server đối chiếu khi có request, refresh token đi kèm để cấp lại access token mới.

```mermaid
erDiagram
    USER ||--o{ SHOP : "sở hữu/quản lý"
    USER ||--o{ SESSION : "đăng nhập"
    SHOP ||--o{ ORDER : "có"
    SHOP ||--o{ CUSTOMER : "có"
    SHOP ||--o{ INVENTORY_ITEM : "có"
    USER ||--o{ OAUTH_APP : "đăng ký (developer)"
    OAUTH_APP ||--o{ AUTHORIZATION_CODE : "phát hành"
    USER ||--o{ AUTHORIZATION_CODE : "được cấp cho (đã login khi approve)"
    OAUTH_APP ||--o{ APP_GRANT : "được cấp quyền"
    USER ||--o{ APP_GRANT : "phê duyệt (shop owner)"
    SHOP ||--o{ APP_GRANT : "dữ liệu được cấp quyền"
    APP_GRANT ||--o{ ACCESS_TOKEN : "sinh ra"
    ACCESS_TOKEN ||--o| REFRESH_TOKEN : "kèm theo"

    USER {
        string id PK
        string email
        string password_hash
        string role "shop_owner | staff | developer"
    }
    SESSION {
        string id PK
        string user_id FK
        string session_token
        datetime created_at
        datetime expires_at
    }
    SHOP {
        string id PK
        string owner_user_id FK
        string name
        datetime created_at
    }
    ORDER {
        string id PK
        string shop_id FK
        string customer_id FK
        decimal total
        string status
    }
    CUSTOMER {
        string id PK
        string shop_id FK
        string name
        string email
    }
    INVENTORY_ITEM {
        string id PK
        string shop_id FK
        string sku
        int quantity
    }
    OAUTH_APP {
        string id PK
        string developer_user_id FK
        string name
        string client_id
        string client_secret_hash "rỗng/không dùng nếu app dùng PKCE"
        string redirect_uris "whitelist"
        datetime created_at
    }
    AUTHORIZATION_CODE {
        string code PK
        string oauth_app_id FK
        string user_id FK
        string shop_id FK
        string scopes "vd read_orders, read_customers, write_inventory"
        string code_challenge "PKCE, cho app không giữ được client_secret an toàn"
        string code_challenge_method
        string redirect_uri
        boolean used
        datetime expires_at "60 giây sau khi phát"
        datetime created_at
    }
    APP_GRANT {
        string id PK
        string oauth_app_id FK
        string user_id FK
        string shop_id FK
        string approved_scopes
        datetime granted_at
        string status "active | revoked"
        datetime revoked_at
    }
    ACCESS_TOKEN {
        string id PK
        string app_grant_id FK
        string token_hash
        string scopes
        datetime expires_at
        datetime revoked_at
    }
    REFRESH_TOKEN {
        string id PK
        string access_token_id FK
        string token_hash
        datetime expires_at
        datetime revoked_at
    }
```
