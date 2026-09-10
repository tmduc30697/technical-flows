# ERD — Enhance (sau khi tối ưu độ trễ end-to-end)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 5 yêu cầu của đề bài — hàng đợi ưu tiên SLA riêng, chuẩn hóa nhiều tỉ lệ khung hình, kiểm tra sơ bộ trước khi vào hàng đợi, tự scale/backpressure, và hủy job khi video bị xóa. So với base, `TRANSCODE_JOB` nay bắt buộc qua `VALIDATION_RESULT` trước khi được enqueue, và có thêm `priority_queue`/`status = cancelled` để phản ánh vòng đời mới.

```mermaid
erDiagram
    USER ||--o{ VIDEO : uploads
    VIDEO ||--o| VALIDATION_RESULT : "checked by"
    VIDEO ||--o| TRANSCODE_JOB : "queued as"
    TRANSCODE_JOB ||--o{ VIDEO_RENDITION : produces
    TRANSCODE_JOB }o--|| WORKER_POOL : "dispatched by"

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

    VALIDATION_RESULT {
        string validation_id PK
        string video_id FK
        boolean passed
        string reason
        datetime checked_at
    }

    TRANSCODE_JOB {
        string job_id PK
        string video_id FK
        string priority_queue
        string status
        datetime queued_at
        datetime completed_at
        datetime cancelled_at
    }

    VIDEO_RENDITION {
        string rendition_id PK
        string job_id FK
        string aspect_ratio
        string storage_path
        string status
    }

    WORKER_POOL {
        string pool_id PK
        int active_workers
        int queue_depth
        string backpressure_state
    }
```
