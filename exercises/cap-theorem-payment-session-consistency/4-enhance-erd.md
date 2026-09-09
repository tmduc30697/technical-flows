# Enhance ERD — sau khi có tunable consistency theo API

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm entity mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `CONSISTENCY_POLICY` (mới) — phân loại rõ endpoint nào AP, endpoint nào CP.
- `ORDER_LOCK` (mới) — khóa CP cho xác nhận thanh toán, đảm bảo chỉ 1 request thắng khi có 2 request trùng.
- `PARTITION_REJECTION_LOG` (mới) — ghi nhận các lần API CP từ chối rõ ràng khi không đủ quorum.
- `STALENESS_ALERT` (mới) — cảnh báo khi độ trễ của API AP vượt ngưỡng bất thường.
- `CONSISTENCY_CONTRACT` (mới) — tài liệu hoá guarantee từng endpoint cho team khác dùng đúng cách.

```mermaid
erDiagram
    CHECKOUT_SESSION ||--o{ SESSION_REPLICA : "replicated as"
    CHECKOUT_SESSION ||--o| ORDER_LOCK : "locked by (khi CP)"
    CONSISTENCY_POLICY ||--o{ PARTITION_REJECTION_LOG : "may reject via"
    CONSISTENCY_POLICY ||--o{ STALENESS_ALERT : "monitored via (khi AP)"
    CONSISTENCY_POLICY ||--|| CONSISTENCY_CONTRACT : "documented as"

    CHECKOUT_SESSION {
        string id PK
        string cart_id
        string status "pending | confirmed"
        datetime updated_at
    }
    SESSION_REPLICA {
        string id PK
        string session_id FK
        string node_id
        string status
        datetime updated_at
    }
    CONSISTENCY_POLICY {
        string id PK
        string endpoint_name
        string mode "AP | CP"
        int quorum_required_for_cp
    }
    ORDER_LOCK {
        string id PK
        string session_id FK
        string holder_request_id
        datetime acquired_at
        datetime expires_at
    }
    PARTITION_REJECTION_LOG {
        string id PK
        string endpoint_name
        datetime attempted_at
        string reason "insufficient_quorum"
    }
    STALENESS_ALERT {
        string id PK
        string endpoint_name
        string node_id
        int staleness_ms
        datetime threshold_exceeded_at
    }
    CONSISTENCY_CONTRACT {
        string id PK
        string endpoint_name
        string guarantee_description
        boolean published_to_consumers
    }
```
