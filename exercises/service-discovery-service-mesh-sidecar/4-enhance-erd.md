# ERD — Enhance (sau khi sidecar có liveness/readiness và fail-open cache)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng yêu cầu đề bài — tách liveness/readiness với nguyên nhân lỗi rõ ràng, đếm số lần readiness pass liên tiếp cho warm-up, backoff khi liên tục fail, và cache snapshot endpoint tại sidecar để fail-open khi mất control plane. So với base, `HEALTH_CHECK_RESULT` nay có `check_type` và `failure_reason` thay vì chỉ 1 kết quả chung.

```mermaid
erDiagram
    SERVICE ||--o{ POD : "has"
    POD ||--|| SIDECAR : "runs"
    SIDECAR ||--o{ HEALTH_CHECK_RESULT : records
    SIDECAR ||--|| ENDPOINT_CACHE_SNAPSHOT : "keeps"

    SERVICE {
        string service_id PK
        string name
    }

    POD {
        string pod_id PK
        string service_id FK
        string ip_address
        string liveness_status
        string readiness_status
        int readiness_consecutive_pass
        datetime scheduled_at
    }

    SIDECAR {
        string sidecar_id PK
        string pod_id FK
        int backoff_level
    }

    HEALTH_CHECK_RESULT {
        string check_id PK
        string sidecar_id FK
        string target_pod_id FK
        string check_type
        string result
        string failure_reason
        datetime checked_at
    }

    ENDPOINT_CACHE_SNAPSHOT {
        string snapshot_id PK
        string sidecar_id FK
        datetime cached_at
        string source
    }
```
