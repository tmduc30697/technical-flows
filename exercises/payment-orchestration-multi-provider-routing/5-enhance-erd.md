# Enhance ERD — Thêm mã tham chiếu, log routing, chuẩn hoá callback và metric dashboard

Đây là **enhance**, mô hình dữ liệu sau khi áp các cơ chế an toàn lên base. So với base: `TRANSACTION` thêm `reference_code` unique (mã tham chiếu gửi kèm mọi request tới provider, dùng để map ngược callback thay vì đoán theo amount+time) và trạng thái `routing` được chuyển bằng update nguyên tử có điều kiện. `PAYMENT_REQUEST` thêm `attempt_number` và trạng thái `verifying`/`cancelled_refunded` cho luồng xác minh trước failover và huỷ/hoàn tiền khi double-success. Thêm mới `PROVIDER_CALLBACK` — chuẩn hoá mọi callback (dù format gốc khác nhau) về cùng 1 mô hình trạng thái nội bộ, map lại `PAYMENT_REQUEST` qua `reference_code`. Thêm mới `ROUTING_LOG` — log đầy đủ lịch sử routing của từng giao dịch. Thêm mới `PROVIDER_METRIC` — số liệu thành công/lỗi real-time theo từng nhà cung cấp phục vụ dashboard và tự động điều chỉnh tỷ lệ routing.

```mermaid
erDiagram
    TRANSACTION ||--o{ PAYMENT_REQUEST : "routed via"
    PROVIDER ||--o{ PAYMENT_REQUEST : handles
    PAYMENT_REQUEST ||--o{ PROVIDER_CALLBACK : "receives"
    TRANSACTION ||--o{ ROUTING_LOG : "tracked by"
    PROVIDER ||--o{ PROVIDER_METRIC : "measured by"

    TRANSACTION {
        string id PK
        decimal amount
        string reference_code UK "mã tham chiếu duy nhất, gửi kèm mọi request tới provider"
        string status "pending | routing | sent_to_provider | success | failed | double_charge_resolved"
    }
    PROVIDER {
        string id PK
        string name
        decimal fee_percent
        string availability_status "up | down"
        int routing_weight
    }
    PAYMENT_REQUEST {
        string id PK
        string transaction_id FK
        string provider_id FK
        int attempt_number "thứ tự thử, 1 = provider đầu tiên, 2 = provider failover..."
        string status "sent | timeout | verifying | success | failed | cancelled_refunded"
        datetime sent_at
    }
    PROVIDER_CALLBACK {
        string id PK
        string payment_request_id FK
        string provider_id FK
        string reference_code_received "map ngược qua mã tham chiếu, không đoán theo amount+time"
        string raw_status "nguyên văn theo format riêng của từng provider"
        string normalized_status "success | failed | pending, đã chuẩn hoá"
        datetime received_at
    }
    ROUTING_LOG {
        string id PK
        string transaction_id FK
        string provider_id FK
        string action "selected | sent | timeout | verified_failed | failover | success | cancelled_due_to_double_success"
        string note
        datetime occurred_at
    }
    PROVIDER_METRIC {
        string id PK
        string provider_id FK
        datetime window_start
        datetime window_end
        int success_count
        int failure_count
        decimal success_rate
    }
```
