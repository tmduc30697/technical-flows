# ERD — Enhance (sau khi có xử lý ưu tiên tốc độ)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 4 yêu cầu của đề bài — ưu tiên xử lý tốc độ cao, tách track theo người nói với đồng bộ chính xác, xử lý recording bị lỗi/dừng giữa chừng, và cô lập dữ liệu theo tổ chức. So với base, `RECORDING` nay gắn `priority_tier` và `org_id` tường minh để cô lập hàng đợi/storage, và có thêm `SPEAKER_TRACK`/`DIARIZATION_RESULT` để tái sử dụng kết quả tách nguồn cho các yêu cầu sau.

```mermaid
erDiagram
    ORGANIZATION ||--o{ MEETING : hosts
    MEETING ||--o{ PARTICIPANT : includes
    MEETING ||--o| RECORDING : "recorded as"
    RECORDING ||--o| PROCESSING_JOB : "processed by"
    RECORDING ||--o| DIARIZATION_RESULT : "diarized into (reusable)"
    DIARIZATION_RESULT ||--o{ SPEAKER_TRACK : contains
    SPEAKER_TRACK }o--|| PARTICIPANT : "attributed to"
    RECORDING ||--o| DELIVERY : "delivered as"

    ORGANIZATION {
        string org_id PK
        string name
        string plan_tier
        string isolation_namespace
    }

    MEETING {
        string meeting_id PK
        string org_id FK
        string title
        datetime started_at
        datetime ended_at
        string status
    }

    PARTICIPANT {
        string participant_id PK
        string meeting_id FK
        string display_name
        string audio_track_ref
    }

    RECORDING {
        string recording_id PK
        string meeting_id FK
        string org_id FK
        string storage_path
        string status
        boolean is_partial
        datetime created_at
    }

    PROCESSING_JOB {
        string job_id PK
        string recording_id FK
        string org_id FK
        string priority_tier
        string status
        datetime queued_at
        datetime completed_at
    }

    DIARIZATION_RESULT {
        string diarization_id PK
        string recording_id FK
        string status
        datetime created_at
    }

    SPEAKER_TRACK {
        string track_id PK
        string diarization_id FK
        string participant_id FK
        string storage_path
        decimal sync_offset_ms
    }

    DELIVERY {
        string delivery_id PK
        string recording_id FK
        string sent_to
        string delivery_status
        boolean is_partial_delivery
        datetime delivered_at
    }
```
