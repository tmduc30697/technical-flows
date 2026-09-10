# ERD — Enhance (sau khi có pipeline chuẩn hóa video)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 4 yêu cầu của đề bài — chuẩn hóa đa dạng input, xử lý bất đồng bộ không chặn đăng tin, kiểm tra ngưỡng chất lượng đầu vào, và thay video không tạo khoảng trống/lẫn lộn. So với base, `LISTING` không còn phụ thuộc video xử lý xong mới publish được; video giờ đi qua `VIDEO_PROCESSING_JOB` và chỉ khi có `VIDEO_RENDITION` sẵn sàng mới thay thế `active_video_id` một cách atomic.

```mermaid
erDiagram
    SELLER ||--o{ LISTING : creates
    LISTING ||--o{ LISTING_IMAGE : has
    LISTING ||--o{ VIDEO_RAW : "uploaded (history)"
    VIDEO_RAW ||--o| VIDEO_PROCESSING_JOB : "queued as"
    VIDEO_PROCESSING_JOB ||--o| VIDEO_QUALITY_CHECK : "gated by"
    VIDEO_PROCESSING_JOB ||--o{ VIDEO_RENDITION : produces
    LISTING ||--o| VIDEO_RENDITION : "active_video (current display)"
    SELLER ||--o{ SELLER_QUEUE_QUOTA : "throttled by"

    SELLER {
        string seller_id PK
        string display_name
        string status
    }

    LISTING {
        string listing_id PK
        string seller_id FK
        string title
        decimal price
        string description
        string status
        string active_video_rendition_id FK
        datetime published_at
    }

    LISTING_IMAGE {
        string image_id PK
        string listing_id FK
        string storage_path
        int position
    }

    VIDEO_RAW {
        string video_id PK
        string listing_id FK
        string storage_path
        string codec
        string resolution
        string aspect_ratio
        string orientation
        datetime uploaded_at
    }

    VIDEO_QUALITY_CHECK {
        string check_id PK
        string video_id FK
        decimal sharpness_score
        decimal brightness_score
        boolean passed
        string reason
    }

    VIDEO_PROCESSING_JOB {
        string job_id PK
        string video_id FK
        string seller_id FK
        string status
        string priority
        datetime queued_at
        datetime completed_at
    }

    VIDEO_RENDITION {
        string rendition_id PK
        string job_id FK
        string storage_path
        string resolution
        string status
        datetime created_at
    }

    SELLER_QUEUE_QUOTA {
        string quota_id PK
        string seller_id FK
        int concurrent_jobs
        int jobs_last_hour
    }
```
