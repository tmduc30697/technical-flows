# Base ERD — Nền tảng gaming trước khi có ingest đa vùng

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái nền tảng streaming gaming **trước khi** có định tuyến ingest đa vùng. Đề bài giả định hệ thống đã stream được (ingest, transcode, phân phối) nhưng chỉ qua **1 điểm ingest trung tâm duy nhất**, chưa phân biệt vùng địa lý của streamer/viewer. Base cần đủ: streamer, điểm ingest trung tâm, phiên live gắn cứng với điểm ingest đó, và viewer xem trực tiếp từ điểm ingest gốc. Chưa có khái niệm vùng địa lý, nhân bản nội dung, failover, hay theo dõi chi phí băng thông theo vùng — những phần đó là enhance.

```mermaid
erDiagram
    STREAMER ||--o{ STREAM_SESSION : starts
    INGEST_ENDPOINT ||--o{ STREAM_SESSION : receives
    STREAM_SESSION ||--o{ VIEWER_CONNECTION : "watched by"

    STREAMER {
        string id PK
        string username
        string location_hint "vi tri uoc luong, chua duoc su dung de dinh tuyen"
    }
    INGEST_ENDPOINT {
        string id PK
        string name "Central-PoP (diem duy nhat)"
    }
    STREAM_SESSION {
        string id PK
        string streamer_id FK
        string ingest_endpoint_id FK "luon la Central-PoP"
        string status "live | ended"
        datetime started_at
        datetime ended_at
    }
    VIEWER_CONNECTION {
        string id PK
        string session_id FK
        string viewer_id
        string viewer_region "chi de tham khao, khong anh huong noi lay noi dung"
    }
```
