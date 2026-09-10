# ERD — Base (trước khi có pipeline transcode tối ưu độ trễ)

Đây là **base**: mô hình dữ liệu suy luận cho app video ngắn *trước khi* có pipeline transcode nhanh, chuẩn hóa, tự scale. Đề bài giả định app đã cho user upload video và có hàng đợi transcode cơ bản — nếu không có sẵn User/Video/TranscodeJob thì các yêu cầu về SLA, chuẩn hóa tỉ lệ khung hình, tự scale, hủy job sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ upload + transcode, không suy diễn thêm follow, like, comment...

```mermaid
erDiagram
    USER ||--o{ VIDEO : uploads
    VIDEO ||--o| TRANSCODE_JOB : "queued as"

    USER {
        string user_id PK
        string display_name
        string status
    }

    VIDEO {
        string video_id PK
        string user_id FK
        string raw_storage_path
        string codec
        string aspect_ratio
        int framerate
        int duration_seconds
        string status
    }

    TRANSCODE_JOB {
        string job_id PK
        string video_id FK
        string status
        datetime queued_at
        datetime completed_at
    }
```
