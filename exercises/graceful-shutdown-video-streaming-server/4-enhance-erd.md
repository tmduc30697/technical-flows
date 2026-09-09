# ERD - Enhance (graceful shutdown cho streaming server)

Đây là **enhance**: mô hình dữ liệu sau khi thêm cơ chế shutdown chuyên biệt cho kết nối streaming dài. So với base, các thay đổi là:

- `STREAM_SESSION` có thêm `estimated_remaining_seconds` (ước tính thời lượng còn lại, dùng để phân loại - yêu cầu 1), `resume_token` và `position_at_disconnect` (để client reconnect đúng vị trí - yêu cầu 1, 3), và `disconnect_reason` (`completed_naturally` / `user_stop` / `shutdown_forced` - yêu cầu 5).
- `GRACE_PERIOD_POLICY` (mới): lưu percentile cấu hình (ví dụ p95) và giá trị grace period tính được từ phân phối `duration_seconds` thực tế của video đang phát - yêu cầu 2.
- `SHUTDOWN_EVENT` (mới): 1 lần shutdown của 1 instance, tham chiếu tới policy đã dùng để tính grace period.
- `SHUTDOWN_BATCH` (mới): gom nhiều `SHUTDOWN_EVENT` xảy ra gần như đồng thời (autoscale scale-in loạt lớn) thành các đợt (batch) có `scheduled_at` giãn cách nhau - yêu cầu 4.
- `DISCONNECT_METRIC` (mới): 1 bản ghi mỗi khi session kết thúc, dùng để tách riêng tỷ lệ ngắt do shutdown khỏi drop-off thông thường - yêu cầu 5.

```mermaid
erDiagram
    CLIENT ||--o{ STREAM_SESSION : opens
    VIDEO ||--o{ STREAM_SESSION : is_played_in
    INSTANCE ||--o{ STREAM_SESSION : serves
    INSTANCE ||--o{ SHUTDOWN_EVENT : undergoes
    SHUTDOWN_EVENT }o--|| GRACE_PERIOD_POLICY : computed_from
    SHUTDOWN_BATCH ||--o{ SHUTDOWN_EVENT : groups
    STREAM_SESSION ||--o{ DISCONNECT_METRIC : records

    CLIENT {
        string id PK
        string device_type
    }

    VIDEO {
        string id PK
        string title
        int duration_seconds
    }

    INSTANCE {
        string id PK
        string status
        string region
        datetime started_at
    }

    STREAM_SESSION {
        string id PK
        string client_id FK
        string video_id FK
        string instance_id FK
        int position_seconds
        int estimated_remaining_seconds
        string resume_token
        int position_at_disconnect
        string disconnect_reason
        string status
        datetime started_at
        datetime ended_at
    }

    GRACE_PERIOD_POLICY {
        string id PK
        int percentile
        int window_days
        int computed_grace_seconds
        datetime updated_at
    }

    SHUTDOWN_EVENT {
        string id PK
        string instance_id FK
        string grace_period_policy_id FK
        string trigger_source
        datetime initiated_at
        int grace_period_seconds
    }

    SHUTDOWN_BATCH {
        string id PK
        int batch_index
        datetime scheduled_at
        string status
    }

    DISCONNECT_METRIC {
        string id PK
        string session_id FK
        string reason
        datetime recorded_at
    }
```
