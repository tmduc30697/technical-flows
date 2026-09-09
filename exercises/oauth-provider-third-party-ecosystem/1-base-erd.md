# Base ERD — Platform e-commerce trước khi có OAuth Provider cho app thứ ba

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài. Đề bài nói enhance sẽ cấp quyền truy cập API dữ liệu shop (đọc order, đọc customer, ghi inventory) cho app bên ngoài — để việc này có nghĩa, base phải đã có sẵn 1 platform e-commerce với Shop, User (chủ shop/nhân viên đăng nhập quản trị shop), và các resource cốt lõi Order/Customer/Inventory mà sau này sẽ bị scope hoá. Base **chưa có** khái niệm OAuth app bên thứ ba, client_id/client_secret, authorization/consent, access token có scope — toàn bộ truy cập dữ liệu hiện chỉ qua session đăng nhập trực tiếp của chủ shop.

```mermaid
erDiagram
    USER ||--o{ SHOP : "sở hữu/quản lý"
    USER ||--o{ SESSION : "đăng nhập"
    SHOP ||--o{ ORDER : "có"
    SHOP ||--o{ CUSTOMER : "có"
    SHOP ||--o{ INVENTORY_ITEM : "có"

    USER {
        string id PK
        string email
        string password_hash
        string role "shop_owner | staff"
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
```
