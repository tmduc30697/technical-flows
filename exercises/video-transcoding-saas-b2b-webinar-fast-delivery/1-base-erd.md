# ERD — Base (trước khi có xử lý ưu tiên tốc độ)

Đây là **base**: mô hình dữ liệu suy luận cho SaaS B2B tổ chức webinar *trước khi* có pipeline xử lý ưu tiên tốc độ và tách nguồn theo người nói. Đề bài giả định hệ thống đã tổ chức được cuộc họp giữa các tổ chức khách hàng và đã ghi lại file recording khi kết thúc — nếu không có sẵn Organization/Meeting/Recording thì yêu cầu "gửi bản ghi nhanh, cô lập theo tổ chức" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ họp + ghi hình, không suy diễn thêm billing, lịch hẹn, CRM...

```mermaid
erDiagram
    ORGANIZATION ||--o{ MEETING : hosts
    MEETING ||--o{ PARTICIPANT : includes
    MEETING ||--o| RECORDING : "recorded as"

    ORGANIZATION {
        string org_id PK
        string name
        string plan_tier
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
        string storage_path
        string status
        datetime created_at
    }
```
