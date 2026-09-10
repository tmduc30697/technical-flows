# ERD — Base (trước khi sidecar có liveness/readiness và fail-open cache)

Đây là **base**: mô hình dữ liệu suy luận cho nền tảng service mesh nội bộ *trước khi* có các yêu cầu nâng cao về sidecar. Đề bài giả định đã có control plane lưu danh sách service/pod, và mỗi pod có sidecar tự discover + health check đơn giản — nếu chưa có POD/SIDECAR/CONTROL_PLANE thì các yêu cầu về warm-up, fail-open, backoff sẽ không có nghĩa. ERD base chỉ có 1 loại health check chung chung.

```mermaid
erDiagram
    SERVICE ||--o{ POD : "has"
    POD ||--|| SIDECAR : "runs"
    SIDECAR ||--o{ HEALTH_CHECK_RESULT : records

    SERVICE {
        string service_id PK
        string name
    }

    POD {
        string pod_id PK
        string service_id FK
        string ip_address
        string status
        datetime scheduled_at
    }

    SIDECAR {
        string sidecar_id PK
        string pod_id FK
    }

    HEALTH_CHECK_RESULT {
        string check_id PK
        string sidecar_id FK
        string target_pod_id FK
        string result
        datetime checked_at
    }
```
