# Enhance ERD — Ingest đa vùng với định tuyến, failover, nhân bản và theo dõi chi phí

Đây là **enhance**, mô hình dữ liệu sau khi áp toàn bộ 5 yêu cầu trong đề bài. So với base, các entity/field mới:
- `REGION` (mới) đại diện 1 khu vực địa lý, chứa nhiều `INGEST_ENDPOINT` — nền tảng cho định tuyến theo địa lý (yêu cầu 1).
- `INGEST_ENDPOINT` thêm `region_id`, `status` (healthy/overloaded/down) và `current_load` — dùng để chọn điểm gần nhất và phát hiện sự cố cần failover (yêu cầu 1, 2).
- `INGEST_ASSIGNMENT` (mới) tách khỏi `STREAM_SESSION`: ghi lại lịch sử session được gán vào endpoint nào, khi nào, vì lý do gì (`nearest`/`failover`/`streamer-moved`) — cho phép đổi điểm ingest giữa buổi mà không tạo session mới (yêu cầu 2, 4).
- `CONTENT_REPLICA` (mới) ghi nhận việc nội dung từ 1 vùng ingest gốc được nhân bản sang vùng khác, kèm độ trễ đo được — phục vụ viewer ở xa (yêu cầu 3).
- `REGION_BANDWIDTH_METRIC` (mới) theo dõi chi phí băng thông và số viewer thực tế theo từng vùng theo thời gian, có cờ bất thường — phục vụ giám sát chi phí (yêu cầu 5).

```mermaid
erDiagram
    REGION ||--o{ INGEST_ENDPOINT : contains
    REGION ||--o{ REGION_BANDWIDTH_METRIC : "measured for"
    STREAMER ||--o{ STREAM_SESSION : starts
    STREAM_SESSION ||--o{ INGEST_ASSIGNMENT : "assigned via"
    INGEST_ENDPOINT ||--o{ INGEST_ASSIGNMENT : "used by"
    STREAM_SESSION ||--o{ CONTENT_REPLICA : "replicated as"
    REGION ||--o{ CONTENT_REPLICA : "replica target"
    STREAM_SESSION ||--o{ VIEWER_CONNECTION : "watched by"

    STREAMER {
        string id PK
        string username
        float last_known_lat
        float last_known_lng
    }
    REGION {
        string id PK
        string name
    }
    INGEST_ENDPOINT {
        string id PK
        string region_id FK
        string status "healthy | overloaded | down"
        int current_load
    }
    STREAM_SESSION {
        string id PK
        string streamer_id FK
        string status "live | ended"
        datetime started_at
        datetime ended_at
    }
    INGEST_ASSIGNMENT {
        string id PK
        string session_id FK
        string ingest_endpoint_id FK
        string reason "nearest | failover | streamer-moved"
        datetime assigned_at
        datetime ended_at
    }
    CONTENT_REPLICA {
        string id PK
        string session_id FK
        string origin_region_id
        string target_region_id FK
        int replication_latency_ms
        datetime replicated_at
    }
    REGION_BANDWIDTH_METRIC {
        string id PK
        string region_id FK
        int viewer_count
        decimal bandwidth_cost
        boolean is_anomalous
        datetime recorded_at
    }
    VIEWER_CONNECTION {
        string id PK
        string session_id FK
        string viewer_id
        string viewer_region
    }
```
