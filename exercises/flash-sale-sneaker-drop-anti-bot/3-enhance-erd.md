# Enhance ERD — Thêm challenge, hàng đợi, idempotency và log rate limit

Đây là **enhance**, mô hình dữ liệu sau khi áp toàn bộ đề bài lên base. So với base, các thay đổi:
- `ORDER` có thêm cột `idempotency_key` (unique) — đáp ứng yêu cầu 3 (chống xử lý trùng khi client tự retry), và có **ràng buộc unique tổng hợp (event_id, user_id)** ghi rõ trong ghi chú entity — đáp ứng yêu cầu 1 (chặn 1 user mua 2 lần ở tầng DB).
- Entity mới `CHALLENGE_TOKEN` — đại diện captcha/challenge phải vượt qua trước khi vào hàng đợi mua — đáp ứng yêu cầu 2 (lớp chống bot độc lập với transaction trừ tồn kho).
- Entity mới `QUEUE_TICKET` — đại diện vé xếp hàng chờ xử lý mua, có trạng thái và thời điểm hết hạn chờ — đáp ứng yêu cầu 4 (trả "hết hàng" trong khoảng thời gian giới hạn thay vì chờ vô thời hạn).
- Entity mới `REQUEST_LOG` — ghi nhận số request theo IP/device fingerprint theo cửa sổ thời gian, có cờ `flagged` để đánh dấu bất thường mà không chặn cứng — đáp ứng yêu cầu 5.

```mermaid
erDiagram
    USER ||--o{ ORDER : places
    EVENT ||--o{ ORDER : "sold in"
    USER ||--o{ CHALLENGE_TOKEN : completes
    EVENT ||--o{ CHALLENGE_TOKEN : "required for"
    USER ||--o{ QUEUE_TICKET : holds
    EVENT ||--o{ QUEUE_TICKET : "queues for"
    QUEUE_TICKET ||--o| ORDER : "resolves to"
    EVENT ||--o{ REQUEST_LOG : "observed during"

    USER {
        string id PK
        string email
        string shipping_address
    }
    EVENT {
        string id PK
        string name
        int total_stock
        int remaining_stock
        datetime starts_at
    }
    ORDER {
        string id PK
        string user_id FK "unique cung event_id - UNIQUE(event_id, user_id)"
        string event_id FK "unique cung user_id - UNIQUE(event_id, user_id)"
        string idempotency_key UK "unique, do client sinh ra"
        string status "pending | paid | failed"
        datetime created_at
    }
    CHALLENGE_TOKEN {
        string id PK
        string user_id FK
        string event_id FK
        string status "issued | verified | expired"
        datetime issued_at
        datetime verified_at
    }
    QUEUE_TICKET {
        string id PK
        string user_id FK
        string event_id FK
        string idempotency_key
        string status "queued | processing | fulfilled | out_of_stock"
        datetime enqueued_at
        datetime expires_at
    }
    REQUEST_LOG {
        string id PK
        string event_id FK
        string ip
        string device_fingerprint
        int request_count
        datetime window_start
        boolean flagged
    }
```
