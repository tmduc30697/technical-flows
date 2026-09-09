# Enhance ERD — sau khi bổ sung load-aware routing, giảm tải có kiểm soát, migration êm, chống race condition và phát hiện suy giảm chất lượng

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có các entity/field mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `EDGE_NODE` (sửa) — thêm `current_viewer_count`, `current_bandwidth_mbps`, `metric_version` (đọc số liệu tải có phiên bản để tránh stale read), `routing_weight` và `quality_score` (yêu cầu 1, 4, 5).
- `LOAD_METRIC_UPDATE` (mới) — mọi lần cập nhật số liệu tải (viewer join/leave/đổi bitrate) được ghi thành sự kiện có `metric_version` tăng dần, dùng cập nhật atomic thay vì đọc-sửa-ghi trực tiếp (yêu cầu 4).
- `ROUTING_DECISION` (mới) — mỗi lần route viewer mới ghi lại `load_snapshot_version` đã dùng, để truy vết và đảm bảo quyết định dựa trên số liệu đủ mới (yêu cầu 1, 4).
- `MIGRATION_EVENT` (mới) — ghi lại việc chuyển một viewer đang xem dở từ edge quá tải sang edge khác kèm `playback_position_seconds` để nối tiếp không gián đoạn (yêu cầu 3).
- `BITRATE_DEGRADATION_EVENT` (mới) — ghi lại việc hạ bitrate có kiểm soát cho một phần viewer trên node quá tải thay vì để tất cả cùng giật (yêu cầu 2).
- `REBUFFER_SAMPLE` (mới) — mẫu tỉ lệ rebuffer từng viewer trên từng edge node, dùng phát hiện suy giảm chất lượng dù node vẫn "healthy" theo health check cơ bản (yêu cầu 5).

```mermaid
erDiagram
    VIDEO_STREAM ||--o{ VIEWER_SESSION : "được xem qua"
    EDGE_NODE ||--o{ VIEWER_SESSION : "phục vụ"
    EDGE_NODE ||--o{ LOAD_METRIC_UPDATE : "nhận cập nhật"
    VIEWER_SESSION ||--o{ ROUTING_DECISION : "có quyết định route"
    VIEWER_SESSION ||--o{ MIGRATION_EVENT : "có thể được chuyển node"
    VIEWER_SESSION ||--o{ BITRATE_DEGRADATION_EVENT : "có thể bị hạ bitrate"
    EDGE_NODE ||--o{ REBUFFER_SAMPLE : "được đo chất lượng"

    VIDEO_STREAM {
        string id PK
        string title
        string status "live | vod"
    }
    EDGE_NODE {
        string id PK
        string region
        string geo_location
        int capacity_max_viewers
        int current_viewer_count
        decimal current_bandwidth_mbps
        int metric_version
        decimal routing_weight
        decimal quality_score
        string health_status "healthy | unhealthy"
    }
    VIEWER_SESSION {
        string id PK
        string video_stream_id FK
        string edge_node_id FK
        string viewer_region
        string current_bitrate
        string status "active | migrated | ended"
        datetime joined_at
    }
    ROUTING_DECISION {
        string id PK
        string viewer_session_id FK
        string candidate_edge_node_id FK
        int latency_estimate_ms
        int load_snapshot_version
        datetime decided_at
    }
    LOAD_METRIC_UPDATE {
        string id PK
        string edge_node_id FK
        string source_event "viewer_join | viewer_leave | bitrate_change"
        int delta_viewer_count
        int resulting_metric_version
        datetime recorded_at
    }
    MIGRATION_EVENT {
        string id PK
        string viewer_session_id FK
        string from_edge_node_id FK
        string to_edge_node_id FK
        int playback_position_seconds
        string triggered_reason "overload"
        datetime migrated_at
    }
    BITRATE_DEGRADATION_EVENT {
        string id PK
        string viewer_session_id FK
        string edge_node_id FK
        string old_bitrate
        string new_bitrate
        datetime applied_at
    }
    REBUFFER_SAMPLE {
        string id PK
        string edge_node_id FK
        string viewer_session_id FK
        decimal rebuffer_ratio
        datetime sampled_at
    }
```
