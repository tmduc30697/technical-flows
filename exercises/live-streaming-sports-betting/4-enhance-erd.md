# Enhance ERD — sau khi bổ sung công bằng độ trễ, đồng bộ overlay, backup ingest, DVR và audit

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có các entity/field mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `INGEST_SOURCE` (sửa) — thêm `role` (primary/backup) và `health_status`, `last_heartbeat_at` để phục vụ failover (yêu cầu 3).
- `FAILOVER_EVENT` (mới) — ghi nhận mỗi lần chuyển từ ingest chính sang ingest dự phòng (yêu cầu 3).
- `STREAM_SEGMENT` (sửa) — thêm `video_pts` (mốc thời gian trình chiếu) làm trục thời gian chung cho toàn hệ thống (nền tảng cho yêu cầu 1 và 2).
- `DELIVERY_LATENCY_PROFILE` (mới) — độ trễ mục tiêu và độ trễ đo được thực tế của từng edge node, dùng để ép các edge cùng phát tại một mốc `video_pts` (yêu cầu 1).
- `EVENT_TIMELINE` (mới) — mốc `video_pts` gắn với từng sự kiện trận đấu thực tế, là điểm neo để đồng bộ overlay (yêu cầu 2).
- `ODDS_UPDATE` (sửa) — thêm `target_display_pts` để client chỉ hiển thị overlay khi player đã decode tới đúng mốc đó (yêu cầu 2).
- `DVR_BUFFER` (mới) — cửa sổ lưu tạm vài chục giây gần nhất theo từng edge, phục vụ time-shift mà không ảnh hưởng luồng chính (yêu cầu 4).
- `AUDIT_LOG` (mới) — log đầy đủ mốc thời gian của mọi sự kiện (ingest, failover, delivery, odds) để đối soát tranh chấp (yêu cầu 5).

```mermaid
erDiagram
    MATCH ||--o{ INGEST_SOURCE : "có primary + backup"
    INGEST_SOURCE ||--o{ FAILOVER_EVENT : "có thể trigger"
    INGEST_SOURCE ||--o{ STREAM_SEGMENT : "sinh ra"
    STREAM_SEGMENT }o--o{ EDGE_NODE : "phân phối qua"
    EDGE_NODE ||--|| DELIVERY_LATENCY_PROFILE : "được theo dõi bởi"
    EDGE_NODE ||--o{ VIEWER_SESSION : "phục vụ"
    EDGE_NODE ||--o{ DVR_BUFFER : "duy trì"
    MATCH ||--o{ EVENT_TIMELINE : "có"
    MATCH ||--o{ ODDS_UPDATE : "có nhiều"
    ODDS_UPDATE }o--|| EVENT_TIMELINE : "neo theo"
    MATCH ||--o{ AUDIT_LOG : "được ghi lại trong"

    MATCH {
        string id PK
        string name
        datetime start_time
        string status "scheduled | live | ended"
    }
    INGEST_SOURCE {
        string id PK
        string match_id FK
        string role "primary | backup"
        string rtmp_url
        string health_status "active | degraded | down"
        datetime last_heartbeat_at
    }
    FAILOVER_EVENT {
        string id PK
        string from_ingest_source_id FK
        string to_ingest_source_id FK
        string reason
        datetime triggered_at
    }
    STREAM_SEGMENT {
        string id PK
        string ingest_source_id FK
        int sequence_number
        bigint video_pts "mốc thời gian trình chiếu chung"
        datetime created_at
    }
    EDGE_NODE {
        string id PK
        string region
        int viewer_capacity
    }
    DELIVERY_LATENCY_PROFILE {
        string edge_node_id PK, FK
        int target_latency_ms
        int measured_latency_ms
        datetime updated_at
    }
    VIEWER_SESSION {
        string id PK
        string edge_node_id FK
        string match_id FK
        datetime joined_at
    }
    DVR_BUFFER {
        string id PK
        string edge_node_id FK
        int window_seconds
        bigint earliest_pts
        bigint latest_pts
    }
    EVENT_TIMELINE {
        string id PK
        string match_id FK
        string event_type "goal | card | kickoff"
        bigint video_pts
        datetime occurred_at
    }
    ODDS_UPDATE {
        string id PK
        string match_id FK
        string event_timeline_id FK
        string event_type "goal | card | odds_change"
        decimal odds_value
        bigint target_display_pts
        datetime created_at
    }
    AUDIT_LOG {
        string id PK
        string match_id FK
        string source_type "ingest | failover | edge_delivery | odds_update"
        string source_ref_id
        bigint video_pts
        datetime wall_clock_time
        datetime recorded_at
    }
```
