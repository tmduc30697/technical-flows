# Enhance ERD — sau khi có canary đánh giá theo business metric

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm entity mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `MODEL` thêm `role`/`traffic_percent` — chạy song song stable/canary thay vì thay thế trực tiếp.
- `BUSINESS_METRIC_WINDOW` (mới) — đo CTR/conversion rate riêng theo từng model, không chỉ latency/lỗi.
- `CANARY_EVALUATION_POLICY` (mới) — quy định thời gian canary tối thiểu (đủ dài qua cả ngày thường lẫn cuối tuần) và ngưỡng business metric để kết luận.
- `TRAINING_DATA_LABEL` (mới) — gắn nhãn request theo model đã serving, tránh feedback loop khi train tiếp.
- `CONTENT_SAFETY_FLAG` + `ROLLBACK_EVENT` (mới) — rollback tức thời khi phát hiện gợi ý bất thường/vi phạm chính sách, không chờ phân tích metric dài hạn.

```mermaid
erDiagram
    MODEL ||--o{ RECOMMENDATION_REQUEST : serves
    MODEL ||--o{ BUSINESS_METRIC_WINDOW : measured
    MODEL ||--o{ ROLLBACK_EVENT : "may trigger on"
    RECOMMENDATION_REQUEST ||--o| TRAINING_DATA_LABEL : "tagged by"
    RECOMMENDATION_REQUEST ||--o| CONTENT_SAFETY_FLAG : "flagged as"

    MODEL {
        string id PK
        string name
        string version
        string role "stable | canary"
        int traffic_percent
        datetime trained_at
        datetime deployed_at
    }
    RECOMMENDATION_REQUEST {
        string id PK
        string user_id
        string model_id FK
        int latency_ms
        boolean error
        boolean clicked
        boolean purchased
        datetime served_at
    }
    BUSINESS_METRIC_WINDOW {
        string id PK
        string model_id FK
        datetime window_start
        datetime window_end
        decimal click_through_rate
        decimal conversion_rate
    }
    CANARY_EVALUATION_POLICY {
        string id PK
        int min_duration_hours "vd 168, đủ 1 tuần"
        decimal business_metric_threshold
        decimal technical_metric_threshold
    }
    TRAINING_DATA_LABEL {
        string id PK
        string recommendation_request_id FK
        string served_by_model_role "stable | canary"
        boolean excluded_from_training
    }
    CONTENT_SAFETY_FLAG {
        string id PK
        string recommendation_request_id FK
        string model_id FK
        string flag_type "irrelevant_recommendation | policy_violation"
        datetime flagged_at
    }
    ROLLBACK_EVENT {
        string id PK
        string model_id FK
        string triggered_by "business_metric_policy | content_safety_flag | manual"
        string reason
        datetime triggered_at
    }
```
