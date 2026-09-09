# Enhance ERD — sau khi có quorum thích ứng theo giai đoạn

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm entity mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `QUORUM_POLICY` (mới) — nhiều mức cấu hình theo ngưỡng tồn kho (trước sale R thấp, gần hết hàng W cao), thay cho 1 `QUORUM_CONFIG` cố định như base.
- `PARTITION_STATUS` (mới) — nhận diện node nào thuộc nhóm minority khi có partition, để từ chối trừ kho ở đó.
- `RECONCILIATION_REPORT` (mới) — đối soát tổng bán ra ghi nhận với tồn kho vật lý thực tế cuối sale.
- `CAPACITY_BENCHMARK` (mới) — đo request/giây chịu được ở từng mode.

```mermaid
erDiagram
    PRODUCT ||--o{ INVENTORY_REPLICA : "replicated as"
    PRODUCT ||--o{ QUORUM_POLICY : "governed by (theo ngưỡng)"
    PRODUCT ||--o| RECONCILIATION_REPORT : "reconciled via"
    INVENTORY_NODE ||--o{ INVENTORY_REPLICA : hosts
    INVENTORY_NODE ||--o{ PARTITION_STATUS : "monitored for"
    QUORUM_POLICY ||--o{ CAPACITY_BENCHMARK : "benchmarked as"

    PRODUCT {
        string id PK
        string name
        int initial_stock
    }
    INVENTORY_NODE {
        string id PK
        string region
    }
    INVENTORY_REPLICA {
        string id PK
        string product_id FK
        string node_id FK
        int stock_value
        datetime last_updated_at
    }
    QUORUM_POLICY {
        string id PK
        string product_id FK
        string mode_name "pre_sale | low_stock_danger"
        decimal stock_threshold_percent
        int read_quorum
        int write_quorum
    }
    PARTITION_STATUS {
        string id PK
        string node_id FK
        string partition_group
        boolean is_majority
        datetime detected_at
    }
    RECONCILIATION_REPORT {
        string id PK
        string product_id FK
        int total_sold_recorded
        int physical_stock_actual
        int discrepancy
        datetime generated_at
    }
    CAPACITY_BENCHMARK {
        string id PK
        string mode_name
        int max_requests_per_second
        datetime measured_at
    }
```
