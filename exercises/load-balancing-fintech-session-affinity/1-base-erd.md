# Base ERD — LB thanh toán trước khi có consistent hashing

Đây là **base**: mô hình dữ liệu suy luận cho load balancer đứng trước service xử lý giao dịch chuyển tiền nhiều bước, ở trạng thái **trước khi** áp consistent hashing. Base chỉ cần đủ: giao dịch nhiều bước được route round-robin không đảm bảo cùng instance, và mỗi instance có thể tạm giữ state trong bộ nhớ (không bền vững, dễ mất) để xử lý OTP nhiều bước — đủ để yêu cầu enhance "có nghĩa" khi so sánh.

```mermaid
erDiagram
    TRANSACTION ||--o{ TRANSACTION_STEP : "có nhiều bước"
    TRANSACTION_STEP }o--|| INSTANCE : "thực thi tại"
    TRANSACTION ||--o| TRANSACTION_STATE_CACHE : "có state tạm (không bền vững)"

    TRANSACTION {
        string id PK
        string user_id
        string transaction_type "transfer | otp_verify"
        string status "pending | completed | failed"
        datetime created_at
    }
    TRANSACTION_STEP {
        string id PK
        string transaction_id FK
        int step_number
        string instance_id FK
        datetime executed_at
    }
    INSTANCE {
        string id PK
        string host
        string status "up | down"
    }
    TRANSACTION_STATE_CACHE {
        string transaction_id PK, FK
        string instance_id FK
        string otp_code
        int retry_count
        datetime cached_at
    }
```
