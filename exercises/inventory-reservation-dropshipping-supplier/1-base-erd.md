# ERD - Base (trước khi có xác nhận thật với nhà cung cấp)

Đây là **base**: mô hình dữ liệu tối thiểu của một sàn dropshipping, nơi tồn kho hiển thị là một cache đồng bộ định kỳ từ API nhà cung cấp, khách đặt mua dựa trên cache đó. Chỉ giữ `Supplier`, `Product` (mang `cached_quantity`), `Customer`, `Order` - đủ để thấy vấn đề gốc: đặt hàng chỉ dựa vào cache có thể lệch với tồn kho thật, và cache có thể bị đồng bộ đè mất thay đổi tạm thời.

```mermaid
erDiagram
    SUPPLIER ||--o{ PRODUCT : supplies
    CUSTOMER ||--o{ ORDER : places
    PRODUCT ||--o{ ORDER : ordered_in

    SUPPLIER {
        string id PK
        string name
        string api_endpoint
    }

    PRODUCT {
        string id PK
        string supplier_id FK
        string title
        int cached_quantity
        datetime last_synced_at
    }

    CUSTOMER {
        string id PK
        string name
    }

    ORDER {
        string id PK
        string product_id FK
        string customer_id FK
        int quantity
        string status
        datetime created_at
    }
```
