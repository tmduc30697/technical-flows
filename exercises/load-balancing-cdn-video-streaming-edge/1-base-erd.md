# Base ERD — CDN video streaming trước khi có load balancing thông minh ở edge

Đây là **base**: mô hình dữ liệu suy luận cho một CDN phục vụ video streaming, ở trạng thái **trước khi** áp các yêu cầu về chọn node theo tải, giảm tải có kiểm soát, di chuyển viewer êm, xử lý race condition số liệu tải, và phát hiện suy giảm chất lượng ngầm. Base chỉ cần đủ: edge node theo khu vực địa lý và viewer session gắn cứng vào một edge node theo quy tắc gần nhất đơn thuần — đủ để các yêu cầu enhance "có nghĩa" khi so sánh.

```mermaid
erDiagram
    VIDEO_STREAM ||--o{ VIEWER_SESSION : "được xem qua"
    EDGE_NODE ||--o{ VIEWER_SESSION : "phục vụ"

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
    }
    VIEWER_SESSION {
        string id PK
        string video_stream_id FK
        string edge_node_id FK
        string viewer_region
        string current_bitrate
        datetime joined_at
    }
```
