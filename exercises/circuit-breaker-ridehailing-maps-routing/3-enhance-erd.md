# Enhance ERD — sau khi có breaker nhạy latency + graceful switch

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 5 nhóm entity mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `CACHED_ROUTE` (mới) — route gần nhất tính thành công, dùng làm fallback tức thời khi API chậm.
- `CIRCUIT_BREAKER_STATE` thêm `latency_threshold_ms` — coi timeout mềm (chậm) như 1 dạng lỗi để tính vào ngưỡng mở breaker.
- `ACTIVE_TRIP_PROVIDER_ASSIGNMENT` + `FALLBACK_PROVIDER_CONFIG` (mới) — graceful switch toàn bộ chuyến đang chạy sang provider dự phòng, không cần khởi động lại app.
- `ROUTE_LATENCY_METRIC` (mới) — đo latency theo từng khu vực địa lý.
- `DRIVER_FEEDBACK` + `BREAKER_FEEDBACK_CORRELATION` (mới) — tương quan thời gian breaker mở với phản hồi tiêu cực từ tài xế.

```mermaid
erDiagram
    TRIP ||--o{ ROUTE_REQUEST : "requests route via"
    TRIP ||--o| CACHED_ROUTE : "has last known"
    TRIP ||--o| ACTIVE_TRIP_PROVIDER_ASSIGNMENT : "assigned provider"
    TRIP ||--o{ DRIVER_FEEDBACK : "may report"
    CIRCUIT_BREAKER_STATE ||--o| FALLBACK_PROVIDER_CONFIG : "falls back via"
    CIRCUIT_BREAKER_STATE ||--o{ ROUTE_LATENCY_METRIC : measured
    CIRCUIT_BREAKER_STATE ||--o{ BREAKER_FEEDBACK_CORRELATION : "correlated with"

    TRIP {
        string id PK
        string driver_id
        string status "active"
    }
    ROUTE_REQUEST {
        string id PK
        string trip_id FK
        int response_time_ms
        string status "success | soft_timeout | error"
        datetime requested_at
    }
    CACHED_ROUTE {
        string trip_id PK
        string route_data
        datetime computed_at
    }
    CIRCUIT_BREAKER_STATE {
        string id PK
        string provider_name
        string state "closed | open | half_open"
        int latency_threshold_ms "vd 2000, coi chậm là lỗi"
        decimal error_rate_threshold
        datetime opened_at
    }
    FALLBACK_PROVIDER_CONFIG {
        string id PK
        string primary_provider
        string fallback_provider
        boolean graceful_switch
    }
    ACTIVE_TRIP_PROVIDER_ASSIGNMENT {
        string trip_id PK
        string provider_in_use "primary | fallback"
        datetime switched_at
    }
    ROUTE_LATENCY_METRIC {
        string id PK
        string provider_name FK
        string region
        datetime window_start
        datetime window_end
        int p95_latency_ms
    }
    DRIVER_FEEDBACK {
        string id PK
        string trip_id FK
        string feedback_type "lost_route | wrong_directions"
        datetime reported_at
    }
    BREAKER_FEEDBACK_CORRELATION {
        string id PK
        string provider_name FK
        datetime breaker_open_window_start
        datetime breaker_open_window_end
        int feedback_count
        datetime generated_at
    }
```
