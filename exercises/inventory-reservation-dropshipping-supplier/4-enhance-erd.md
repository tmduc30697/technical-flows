# ERD - Enhance (xác nhận thật với nhà cung cấp + xử lý lệch tồn kho)

Đây là **enhance**: mô hình dữ liệu sau khi thêm bước xác nhận thật với nhà cung cấp và các cơ chế xử lý lệch tồn kho. So với base, các thay đổi là:

- `PRODUCT` có thêm `reserved_count` (giữ tạm khi order đang chờ xác nhận, tách khỏi `cached_quantity` để sync không đè mất - yêu cầu 3) và `version` (dùng cho update nguyên tử khi trừ cache và khi merge sync - yêu cầu 1, 3).
- `ORDER` có thêm `status` mở rộng (`pending_confirmation` / `confirmed` / `cancelled_out_of_stock` / `cancelled_timeout`), `cancel_reason`, `refunded_at` - phản ánh kết quả xác nhận thật với nhà cung cấp (yêu cầu 1, 2, 4).
- `SUPPLIER_CONFIRMATION_ATTEMPT` (mới): mỗi lần gọi API xác nhận đặt hàng thật, gồm `attempt_number`, `requested_at`, `responded_at`, `result` - phục vụ chính sách retry/timeout (yêu cầu 4).
- `SUPPLIER_DISCREPANCY_ALERT` (mới): tổng hợp tỷ lệ hủy đơn do lệch tồn kho theo từng nhà cung cấp trong 1 khung thời gian, kèm mốc `triggered_at` khi vượt ngưỡng (yêu cầu 5). `SUPPLIER` có thêm `status` (`active` / `hidden`) để team vận hành tạm ẩn sản phẩm khi cảnh báo xảy ra.

```mermaid
erDiagram
    SUPPLIER ||--o{ PRODUCT : supplies
    CUSTOMER ||--o{ ORDER : places
    PRODUCT ||--o{ ORDER : ordered_in
    ORDER ||--o{ SUPPLIER_CONFIRMATION_ATTEMPT : has_attempts
    SUPPLIER ||--o{ SUPPLIER_DISCREPANCY_ALERT : monitored_by

    SUPPLIER {
        string id PK
        string name
        string api_endpoint
        string status
    }

    PRODUCT {
        string id PK
        string supplier_id FK
        string title
        int cached_quantity
        int reserved_count
        int version
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
        string cancel_reason
        datetime created_at
        datetime refunded_at
    }

    SUPPLIER_CONFIRMATION_ATTEMPT {
        string id PK
        string order_id FK
        int attempt_number
        datetime requested_at
        datetime responded_at
        string result
    }

    SUPPLIER_DISCREPANCY_ALERT {
        string id PK
        string supplier_id FK
        datetime window_start
        datetime window_end
        int cancel_count
        int total_orders
        float cancel_rate
        datetime triggered_at
    }
```
