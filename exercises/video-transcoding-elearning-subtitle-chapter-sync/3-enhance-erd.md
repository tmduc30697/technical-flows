# ERD — Enhance (sau khi đồng bộ timestamp phụ đề/chapter qua pipeline)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng yêu cầu đề bài — pipeline transcode nhiều bước với từng `TRANSFORM_STEP` được ghi nhận để tính lại timestamp, ánh xạ mốc gốc sang mốc đã transcode cho chapter marker, pipeline phụ đề/chapter chạy song song độc lập với pipeline video chính, và versioning cho phụ đề để không làm gián đoạn học viên đang xem dở. So với base, `CHAPTER_MARKER` nay có cả `original_timestamp_seconds` lẫn `transcoded_timestamp_seconds`, còn `SUBTITLE_FILE` trở thành nhiều `SUBTITLE_VERSION` áp dụng dần theo lượt xem tiếp theo.

```mermaid
erDiagram
    COURSE ||--o{ LECTURE_VIDEO : contains
    LECTURE_VIDEO ||--|| TRANSCODE_JOB : "processed by"
    TRANSCODE_JOB ||--o{ TRANSFORM_STEP : "consists of"
    TRANSCODE_JOB ||--o{ RENDITION : produces
    LECTURE_VIDEO ||--o{ CHAPTER_MARKER : "marked with"
    LECTURE_VIDEO ||--o{ SUBTITLE_TRACK : "has"
    SUBTITLE_TRACK ||--o{ SUBTITLE_VERSION : "versioned as"
    LECTURE_VIDEO ||--|| SUBTITLE_SYNC_JOB : "processed independently by"
    STUDENT_VIEW_SESSION ||--|| SUBTITLE_VERSION : "pinned to"

    COURSE {
        string course_id PK
        string title
        string instructor_id
    }

    LECTURE_VIDEO {
        string video_id PK
        string course_id FK
        string raw_upload_url
        int original_duration_seconds
        int transcoded_duration_seconds
        string status
    }

    TRANSCODE_JOB {
        string job_id PK
        string video_id FK
        string status
        datetime started_at
        datetime completed_at
    }

    TRANSFORM_STEP {
        string step_id PK
        string job_id FK
        string step_type
        decimal offset_seconds
        decimal scale_factor
        int step_order
    }

    RENDITION {
        string rendition_id PK
        string job_id FK
        string resolution
        string file_url
    }

    CHAPTER_MARKER {
        string marker_id PK
        string video_id FK
        string title
        int original_timestamp_seconds
        int transcoded_timestamp_seconds
    }

    SUBTITLE_TRACK {
        string track_id PK
        string video_id FK
        string language
    }

    SUBTITLE_VERSION {
        string version_id PK
        string track_id FK
        string file_url
        int version_number
        datetime effective_from
        boolean is_current
    }

    SUBTITLE_SYNC_JOB {
        string sync_job_id PK
        string video_id FK
        string status
        datetime completed_at
    }

    STUDENT_VIEW_SESSION {
        string session_id PK
        string video_id FK
        string student_id
        string subtitle_version_id FK
        datetime started_at
    }
```
