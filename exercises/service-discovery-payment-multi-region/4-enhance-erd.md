# ERD — Enhance (sau khi có service discovery đa vùng)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng yêu cầu đề bài — metadata region cho mỗi instance, phân biệt lỗi tạm thời với lỗi thật (tránh flapping), cross-region failover kèm alert và đánh dấu request, và theo dõi trạng thái đồng bộ/partition giữa hai registry. So với base, `INSTANCE` nay có thêm `region_id`, và mọi quyết định route phải đi qua các entity mới này thay vì chỉ dựa trên status nhị phân.

```mermaid
erDiagram
    REGION ||--o{ INSTANCE : hosts
    SERVICE ||--o{ INSTANCE : has
    INSTANCE ||--o{ HEALTH_CHECK_LOG : "checked by"
    REGION ||--o{ FAILOVER_EVENT : "source of"
    FAILOVER_EVENT ||--o| ALERT : triggers
    REGION ||--o{ REGISTRY_SYNC_STATE : "syncs via"

    SERVICE {
        string service_id PK
        string name
    }

    REGION {
        string region_id PK
        string name
        string registry_endpoint
    }

    INSTANCE {
        string instance_id PK
        string service_id FK
        string region_id FK
        string host
        int port
        string status
        datetime registered_at
        datetime last_heartbeat_at
    }

    HEALTH_CHECK_LOG {
        string check_id PK
        string instance_id FK
        string result
        boolean is_transient
        datetime checked_at
    }

    FAILOVER_EVENT {
        string event_id PK
        string service_id FK
        string from_region_id FK
        string to_region_id FK
        string reason
        datetime triggered_at
    }

    ALERT {
        string alert_id PK
        string event_id FK
        string severity
        string acknowledged_by
        datetime acknowledged_at
    }

    REGISTRY_SYNC_STATE {
        string sync_id PK
        string region_id FK
        string peer_region_id FK
        datetime last_synced_at
        int sync_lag_ms
        string partition_status
    }
```
