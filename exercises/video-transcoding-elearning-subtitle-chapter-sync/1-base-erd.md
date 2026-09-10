# ERD — Base (trước khi đồng bộ timestamp phụ đề/chapter qua pipeline)

Đây là **base**: mô hình dữ liệu suy luận cho nền tảng e-learning *trước khi* có yêu cầu đồng bộ timestamp qua nhiều bước transcode. Đề bài giả định nền tảng đã có video bài giảng, transcode ra nhiều độ phân giải, và giảng viên đã có thể gắn phụ đề/chapter marker — nếu chưa có các entity này thì việc "timestamp bị lệch sau khi qua pipeline" sẽ không có nghĩa. Ở base, quá trình transcode chỉ là 1 bước đơn giản, và timestamp phụ đề/chapter được giữ nguyên số giây gốc, không tính lại (đây chính là nguồn gốc gây lệch mà enhance phải xử lý). Giả định dữ liệu lưu quan hệ 1-nhiều đơn giản (SQL), phù hợp quy mô 1 platform e-learning.

```mermaid
erDiagram
    COURSE ||--o{ LECTURE_VIDEO : contains
    LECTURE_VIDEO ||--|| TRANSCODE_JOB : "processed by"
    TRANSCODE_JOB ||--o{ RENDITION : produces
    LECTURE_VIDEO ||--o{ SUBTITLE_FILE : "attached with"
    LECTURE_VIDEO ||--o{ CHAPTER_MARKER : "marked with"

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
        string status
    }

    TRANSCODE_JOB {
        string job_id PK
        string video_id FK
        string status
        datetime started_at
        datetime completed_at
    }

    RENDITION {
        string rendition_id PK
        string job_id FK
        string resolution
        string file_url
    }

    SUBTITLE_FILE {
        string subtitle_id PK
        string video_id FK
        string language
        string file_url
    }

    CHAPTER_MARKER {
        string marker_id PK
        string video_id FK
        string title
        int timestamp_seconds
    }
```
