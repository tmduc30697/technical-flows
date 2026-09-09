# Base ERD — Nền tảng phát sóng thể thao trước khi có yêu cầu công bằng độ trễ

Đây là **base**: mô hình dữ liệu suy luận cho một nền tảng live streaming thể thao có hiển thị tỷ lệ cá cược, ở trạng thái **trước khi** áp các yêu cầu về đồng nhất độ trễ, đồng bộ overlay, backup ingest, DVR và audit log. Base chỉ cần đủ: một nguồn ingest duy nhất, pipeline transcode/phân phối qua edge node, viewer session, và luồng cập nhật tỷ lệ cá cược độc lập với video — đủ để các yêu cầu enhance "có nghĩa" khi so sánh.

```mermaid
erDiagram
    MATCH ||--|| INGEST_SOURCE : "phát từ"
    MATCH ||--o{ ODDS_UPDATE : "có nhiều"
    INGEST_SOURCE ||--o{ STREAM_SEGMENT : "sinh ra"
    STREAM_SEGMENT }o--o{ EDGE_NODE : "phân phối qua"
    EDGE_NODE ||--o{ VIEWER_SESSION : "phục vụ"

    MATCH {
        string id PK
        string name
        datetime start_time
        string status "scheduled | live | ended"
    }
    INGEST_SOURCE {
        string id PK
        string match_id FK
        string rtmp_url
        string status "active | down"
    }
    STREAM_SEGMENT {
        string id PK
        string ingest_source_id FK
        int sequence_number
        datetime created_at
    }
    EDGE_NODE {
        string id PK
        string region
        int viewer_capacity
    }
    VIEWER_SESSION {
        string id PK
        string edge_node_id FK
        string match_id FK
        datetime joined_at
    }
    ODDS_UPDATE {
        string id PK
        string match_id FK
        string event_type "goal | card | odds_change"
        decimal odds_value
        datetime created_at
    }
```
