# ERD - Enhance (có distributed lock điều phối deploy đa vùng)

Đây là trạng thái **enhance**: sau khi áp đề bài, giữ nguyên `REGION`, `PIPELINE`, `DEPLOY_JOB` từ base, thêm mới:
- `DISTRIBUTED_LOCK`: đại diện cho lock toàn cục đặt trên coordinator chịu lỗi cao (ZooKeeper/etcd ensemble), có `lease_ttl_seconds`/`expires_at` để tự hết hạn nếu pipeline treo, và `fencing_token` để region không thể "tự cho là an toàn" nếu token không khớp - đáp ứng yêu cầu 1, 2 và 4.
- `LOCK_COORDINATOR_NODE`: các node trong ensemble/cluster (ZK hoặc etcd), độc lập với mọi region, dùng để thể hiện tính chịu lỗi cao (quorum) của coordinator - đáp ứng yêu cầu 1.
- `DEPLOY_QUEUE_ENTRY`: hàng đợi pipeline đang chờ lock, có `max_wait_seconds` và `status` để xử lý timeout khi chờ quá lâu - đáp ứng yêu cầu 5.
- `AUDIT_LOG_ENTRY`: ghi lại toàn bộ vòng đời của lock và deploy (region nào, ai trigger, lúc nào) phục vụ rollback/điều tra - đáp ứng yêu cầu 3.
- `ALERT`: cảnh báo cho người vận hành khi lock bị treo quá hạn, khi coordinator mất kết nối, hoặc khi queue timeout - đáp ứng yêu cầu 2, 4 và 5.

```mermaid
erDiagram
    REGION ||--o{ PIPELINE : "chạy"
    PIPELINE ||--o{ DEPLOY_JOB : "kích hoạt"
    PIPELINE ||--o{ DEPLOY_QUEUE_ENTRY : "xếp hàng chờ lock"
    DEPLOY_QUEUE_ENTRY }o--|| DISTRIBUTED_LOCK : "chờ cấp"
    DEPLOY_JOB ||--o| DISTRIBUTED_LOCK : "giữ (nếu đang deploy)"
    DISTRIBUTED_LOCK ||--o{ AUDIT_LOG_ENTRY : "sinh ra sự kiện"
    LOCK_COORDINATOR_NODE ||--o{ DISTRIBUTED_LOCK : "lưu trữ, đồng thuận"
    DISTRIBUTED_LOCK ||--o{ ALERT : "kích hoạt khi bất thường"
    REGION ||--o{ AUDIT_LOG_ENTRY : "được ghi log theo"

    REGION {
        string region_id PK
        string name
        string endpoint_url
    }
    PIPELINE {
        string pipeline_id PK
        string region_id FK
        string name
        string status
    }
    DEPLOY_JOB {
        string job_id PK
        string pipeline_id FK
        string triggered_by
        datetime triggered_at
        string version
        string status
    }
    DISTRIBUTED_LOCK {
        string lock_id PK
        string resource_name
        string holder_pipeline_id FK
        string holder_region_id FK
        string fencing_token
        datetime acquired_at
        int lease_ttl_seconds
        datetime expires_at
        string status
    }
    LOCK_COORDINATOR_NODE {
        string node_id PK
        string cluster_type
        string role
        string health_status
    }
    DEPLOY_QUEUE_ENTRY {
        string entry_id PK
        string pipeline_id FK
        string region_id FK
        datetime enqueued_at
        int max_wait_seconds
        string status
    }
    AUDIT_LOG_ENTRY {
        string audit_id PK
        string lock_id FK
        string region_id FK
        string pipeline_id FK
        string action
        string actor
        datetime occurred_at
        string detail
    }
    ALERT {
        string alert_id PK
        string lock_id FK
        string region_id FK
        string alert_type
        datetime created_at
        boolean acknowledged
    }
```
