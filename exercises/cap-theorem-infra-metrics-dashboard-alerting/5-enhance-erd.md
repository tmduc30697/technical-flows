# Enhance ERD — sau khi tách quorum theo mục đích + xử lý partition/backfill

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm entity mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `QUORUM_POLICY` (mới) — tách riêng cấu hình theo mục đích (`dashboard` dùng R thấp, `alert` dùng R cao/đọc node vừa ghi), thay cho 1 config chung như base.
- `PARTITION_WRITE_POLICY` (mới) — quyết định rõ chấp nhận ghi phía minority để ưu tiên availability, và cách merge khi partition hàn lại theo timestamp agent gửi.
- `BACKFILL_EVALUATION` (mới) — đánh giá lại ngưỡng cho dữ liệu gửi bù trễ, kích hoạt cảnh báo hồi tố nếu cần.
- `ALERT_LATENCY_METRIC` + `DASHBOARD_STALENESS_METRIC` (mới) — đo end-to-end alert latency và tỉ lệ stale, tách riêng theo từng luồng.

```mermaid
erDiagram
    SERVER ||--o{ METRIC_SAMPLE : reports
    TIME_SERIES_NODE ||--o{ METRIC_SAMPLE : stores
    SERVER ||--o{ QUORUM_POLICY : "governed by (theo mục đích)"
    TIME_SERIES_NODE ||--o{ PARTITION_WRITE_POLICY : "applies during partition"
    METRIC_SAMPLE ||--o| BACKFILL_EVALUATION : "evaluated by (nếu gửi trễ)"
    SERVER ||--o{ ALERT_LATENCY_METRIC : measures
    SERVER ||--o{ DASHBOARD_STALENESS_METRIC : measures

    SERVER {
        string id PK
        string cluster_id
    }
    METRIC_SAMPLE {
        string id PK
        string server_id FK
        string node_id FK
        string metric_type "cpu | memory | disk | latency"
        decimal value
        datetime sample_timestamp
        datetime received_at
        boolean is_backfill
    }
    TIME_SERIES_NODE {
        string id PK
        string region
    }
    QUORUM_POLICY {
        string id PK
        string purpose "dashboard | alert"
        int read_quorum
        boolean read_from_latest_writer_node
    }
    PARTITION_WRITE_POLICY {
        string id PK
        boolean accept_minority_writes
        string conflict_resolution "by_agent_send_timestamp"
    }
    BACKFILL_EVALUATION {
        string id PK
        string metric_sample_id FK
        boolean breached_threshold_at_sample_time
        boolean retroactive_alert_triggered
        datetime evaluated_at
    }
    ALERT_LATENCY_METRIC {
        string id PK
        string server_id FK
        datetime breach_at
        datetime alert_triggered_at
        int latency_ms
    }
    DASHBOARD_STALENESS_METRIC {
        string id PK
        string server_id FK
        datetime requested_at
        datetime data_as_of
        int staleness_ms
    }
```
