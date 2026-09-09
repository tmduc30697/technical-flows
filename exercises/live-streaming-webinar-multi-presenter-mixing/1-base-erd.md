# Base ERD — Webinar nhiều presenter trước khi có bù trừ độ trễ và crossfade

Đây là **base**: mô hình dữ liệu suy luận cho một nền tảng webinar có nhiều presenter luân phiên trình bày, ở trạng thái **trước khi** áp các yêu cầu về crossfade mượt, bù trừ độ trễ mạng, failover khi rớt kết nối, ưu tiên đúng nguồn audio, và ghi lại chính xác trình tự chuyển đổi. Base chỉ cần đủ: presenter với ingest riêng, lệnh switch đơn giản do host chủ động bấm, và bản ghi output thô — đủ để các yêu cầu enhance "có nghĩa" khi so sánh.

```mermaid
erDiagram
    WEBINAR ||--o{ PRESENTER : "có nhiều"
    PRESENTER ||--|| INGEST_STREAM : "phát luồng"
    WEBINAR ||--|| OUTPUT_STREAM : "sinh ra"
    WEBINAR ||--o{ SWITCH_COMMAND : "có nhiều"
    SWITCH_COMMAND }o--|| PRESENTER : "nhắm tới"
    OUTPUT_STREAM ||--o{ VIEWER_SESSION : "phục vụ"

    WEBINAR {
        string id PK
        string title
        datetime started_at
        string status "scheduled | live | ended"
    }
    PRESENTER {
        string id PK
        string webinar_id FK
        string name
        string role "presenter | host"
        string connection_status "connected | disconnected"
    }
    INGEST_STREAM {
        string id PK
        string presenter_id FK
        string type "webcam | screen_share"
        string audio_mic_status "muted | unmuted"
    }
    SWITCH_COMMAND {
        string id PK
        string webinar_id FK
        string target_presenter_id FK
        datetime issued_at
        string issued_by
    }
    OUTPUT_STREAM {
        string id PK
        string webinar_id FK
        string current_active_presenter_id FK
    }
    VIEWER_SESSION {
        string id PK
        string webinar_id FK
        datetime joined_at
    }
```
