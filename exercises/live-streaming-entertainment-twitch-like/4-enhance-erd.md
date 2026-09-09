# Enhance ERD — Streaming platform với xác thực key, ABR, chống gián đoạn, VOD, chịu tải đột biến

Đây là **enhance**, mô hình dữ liệu sau khi áp toàn bộ 5 yêu cầu trong đề bài. So với base, các entity/field mới:
- `STREAM_KEY` thêm `is_revoked` và `last_validated_at` — ingest kiểm tra trước khi nhận luồng (yêu cầu 1).
- `STREAM_SESSION.status` mở rộng thêm giá trị `interrupted`, cùng `interrupted_at` và `reconnect_deadline` — phân biệt "gián đoạn tạm thời" với "kết thúc hẳn", cho phép khôi phục đúng session (yêu cầu 3).
- `RENDITION` (mới) đại diện 1 mức bitrate (nguồn/trung/thấp) được transcode từ 1 `STREAM_SESSION` — hỗ trợ ABR (yêu cầu 2).
- `VIDEO_SEGMENT` (mới) gắn với `RENDITION`, có `storage_status` và `persisted_at` để đảm bảo không mất segment khi worker transcode sự cố, phục vụ ghép VOD sau (yêu cầu 4).
- `EDGE_NODE` (mới) và `VIEWER_CONNECTION.edge_node_id` — mô hình hoá việc viewer được phân bổ vào các edge phân tán, phục vụ chịu tải đột biến (yêu cầu 5).

```mermaid
erDiagram
    STREAMER ||--o{ STREAM_KEY : owns
    STREAMER ||--o{ STREAM_SESSION : starts
    STREAM_SESSION ||--o{ RENDITION : "transcoded into"
    RENDITION ||--o{ VIDEO_SEGMENT : "produces"
    STREAM_SESSION ||--o{ VIEWER_CONNECTION : "watched by"
    EDGE_NODE ||--o{ VIEWER_CONNECTION : serves

    STREAMER {
        string id PK
        string username
    }
    STREAM_KEY {
        string id PK
        string streamer_id FK
        string key_value
        boolean is_revoked
        datetime last_validated_at
    }
    STREAM_SESSION {
        string id PK
        string streamer_id FK
        string status "live | interrupted | ended"
        datetime started_at
        datetime interrupted_at
        datetime reconnect_deadline
        datetime ended_at
    }
    RENDITION {
        string id PK
        string session_id FK
        string bitrate_level "source | mid | low"
        string transcode_worker_id
    }
    VIDEO_SEGMENT {
        string id PK
        string rendition_id FK
        int sequence_no
        string storage_status "pending | persisted | lost"
        datetime persisted_at
    }
    EDGE_NODE {
        string id PK
        string region
        int current_viewer_count
        string status "healthy | overloaded"
    }
    VIEWER_CONNECTION {
        string id PK
        string session_id FK
        string edge_node_id FK
        string viewer_id
        datetime connected_at
    }
```
