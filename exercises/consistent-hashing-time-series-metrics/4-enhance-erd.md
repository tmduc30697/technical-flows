# Enhance ERD — Partition kết hợp hash(service) + time bucket, kèm archive và hot-partition splitting

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, `PARTITION` được đổi khóa phân mảnh sang composite (`hash_range` + `time_bucket`) thay vì chỉ theo thời gian, và có thêm 4 entity mới, ứng trực tiếp với các yêu cầu trong đề bài:

- `PARTITION` thêm `hash_range` (bucket của `hash(service_id)`) và `is_dedicated` — đáp ứng yêu cầu 1 (composite key hash(service)+time bucket, tránh hot partition).
- `CLUSTER_CAPACITY_CONFIG` (mới) — lưu số node/partition cần thiết để đạt throughput mục tiêu, đáp ứng yêu cầu 4.
- `ARCHIVE_JOB` (mới) — theo dõi việc nén/di chuyển partition cũ sang cold storage mà không chặn ghi mới, đáp ứng yêu cầu 3.
- `SERVICE_LOAD_STATS` và `PARTITION_SPLIT_EVENT` (mới) — phát hiện service gây lệch tải và ghi nhận sự kiện tách partition riêng cho service đó, đáp ứng yêu cầu 5.

```mermaid
erDiagram
    SERVICE ||--o{ METRIC : emits
    METRIC ||--o{ METRIC_POINT : has
    PARTITION ||--o{ METRIC_POINT : stores
    SERVICE ||--o{ SERVICE_LOAD_STATS : "monitored via"
    PARTITION ||--o{ PARTITION_SPLIT_EVENT : "may split into"
    PARTITION ||--o{ ARCHIVE_JOB : "archived by"
    CLUSTER_CAPACITY_CONFIG ||--o{ PARTITION : sizes

    SERVICE {
        string id PK
        string name
    }
    METRIC {
        string id PK
        string service_id FK
        string metric_name
    }
    METRIC_POINT {
        string id PK
        string metric_id FK
        datetime ts
        double value
    }
    PARTITION {
        string id PK
        string hash_range "bucket của hash(service_id)"
        string time_bucket
        string node_id
        boolean is_dedicated "true nếu tách riêng cho 1 service hot"
    }
    CLUSTER_CAPACITY_CONFIG {
        string id PK
        int target_points_per_sec
        int per_node_capacity_pps
        int required_node_count
        int required_partition_count
    }
    ARCHIVE_JOB {
        string id PK
        string partition_id FK
        string status "pending|archiving|done"
        string cold_storage_uri
        datetime archived_at
    }
    SERVICE_LOAD_STATS {
        string id PK
        string service_id FK
        string window
        int points_per_sec
        boolean is_hot
    }
    PARTITION_SPLIT_EVENT {
        string id PK
        string source_partition_id FK
        string hot_service_id FK
        string new_partition_id FK
        datetime triggered_at
    }
```
