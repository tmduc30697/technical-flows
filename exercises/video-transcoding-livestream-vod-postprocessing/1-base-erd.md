# ERD — Base (trước khi có hậu xử lý VOD)

Đây là **base**: mô hình dữ liệu suy luận cho nền tảng livestream *trước khi* có pipeline hậu xử lý VOD. Đề bài giả định hệ thống đã phát live, đã ghi lại các segment trong lúc live, và đã có cơ chế đánh dấu mốc thời gian quan trọng (donate, highlight) — nếu không có sẵn các entity này thì "ghép segment thành VOD, giữ đúng mốc thời gian" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ ghi hình + đánh dấu sự kiện, không suy diễn thêm các module không liên quan (chat, subscription, monetization...).

```mermaid
erDiagram
    STREAMER ||--o{ STREAM : hosts
    STREAM ||--o{ SEGMENT : "recorded as"
    STREAM ||--o{ LIVE_EVENT : "marked during"

    STREAMER {
        string streamer_id PK
        string display_name
        string status
    }

    STREAM {
        string stream_id PK
        string streamer_id FK
        datetime started_at
        datetime ended_at
        string status
    }

    SEGMENT {
        string segment_id PK
        string stream_id FK
        int sequence_number
        string storage_path
        datetime captured_at
        string status
    }

    LIVE_EVENT {
        string event_id PK
        string stream_id FK
        string type
        datetime occurred_at
        string metadata
    }
```
