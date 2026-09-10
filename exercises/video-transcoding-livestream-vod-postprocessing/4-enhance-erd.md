# ERD — Enhance (sau khi có hậu xử lý VOD)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 5 yêu cầu của đề bài — ghép segment thành file liên tục (xử lý thiếu/lỗi), transcode lại chất lượng cao theo từng đoạn (chunked), ánh xạ lại mốc thời gian của `LIVE_EVENT` sang VOD, và dọn raw recording sau khi VOD phát được. So với base, `LIVE_EVENT` không đổi cấu trúc nhưng nay có thêm `vod_offset` để trỏ đúng vị trí trên VOD đã ghép/transcode (khác với `occurred_at` gốc trên live).

```mermaid
erDiagram
    STREAMER ||--o{ STREAM : hosts
    STREAM ||--o{ SEGMENT : "recorded as"
    STREAM ||--o{ LIVE_EVENT : "marked during"
    STREAM ||--o| VOD : "post-processed into"
    VOD ||--o{ VOD_CHUNK : "processed as"
    VOD_CHUNK ||--o{ SEGMENT : "assembled from"
    LIVE_EVENT }o--|| VOD : "remapped onto"

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
        decimal vod_offset_seconds
        string metadata
    }

    VOD {
        string vod_id PK
        string stream_id FK
        string status
        string raw_recording_path
        boolean raw_cleaned_up
        datetime available_at
    }

    VOD_CHUNK {
        string chunk_id PK
        string vod_id FK
        int chunk_index
        int segment_count
        string transcode_status
        string quality_profile
        string output_path
    }
```
