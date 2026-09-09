# Enhance ERD — sau khi bổ sung crossfade, bù trừ độ trễ, failover, ưu tiên audio và ghi log chuyển đổi

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có các entity/field mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `INGEST_STREAM` (sửa) — thêm `network_latency_ms`, `last_frame_received_at` để biết độ trễ thực tế của từng presenter (nền tảng cho yêu cầu 2).
- `SYNC_BUFFER` (mới) — bộ đệm bù trừ riêng cho từng ingest, quy về một mốc thời gian chung trước khi mixer ghép (yêu cầu 2).
- `TRANSITION_EVENT` (mới) — ghi nhận từng lần chuyển đổi thực tế giữa hai presenter, gồm thời điểm lệnh phát ra, thời điểm áp dụng thật, kiểu chuyển (cut/crossfade) — dùng để cả mixer chờ đúng thời điểm chuyển mượt (yêu cầu 1) lẫn phục vụ ghi lại đúng trình tự khi phát lại (yêu cầu 5).
- `DISCONNECT_EVENT` (mới) — phát hiện presenter rớt kết nối và nguồn được chuyển sang thay thế (yêu cầu 3).
- `AUDIO_MIX_DECISION` (mới) — mỗi thời điểm chỉ có một presenter được coi là nguồn audio chính, các nguồn khác bị mute dù đang bật mic (yêu cầu 4).
- `RECORDING` (mới) — bản ghi chính thức, liên kết với toàn bộ `TRANSITION_EVENT` để tái hiện đúng trình tự khi phát lại (yêu cầu 5).

```mermaid
erDiagram
    WEBINAR ||--o{ PRESENTER : "có nhiều"
    PRESENTER ||--|| INGEST_STREAM : "phát luồng"
    INGEST_STREAM ||--|| SYNC_BUFFER : "được bù trừ bởi"
    WEBINAR ||--|| OUTPUT_STREAM : "sinh ra"
    WEBINAR ||--o{ SWITCH_COMMAND : "có nhiều"
    SWITCH_COMMAND }o--|| PRESENTER : "nhắm tới"
    SWITCH_COMMAND ||--|| TRANSITION_EVENT : "dẫn tới"
    PRESENTER ||--o{ DISCONNECT_EVENT : "có thể gây ra"
    WEBINAR ||--o{ AUDIO_MIX_DECISION : "có nhiều"
    OUTPUT_STREAM ||--o{ VIEWER_SESSION : "phục vụ"
    WEBINAR ||--|| RECORDING : "sinh ra"
    RECORDING ||--o{ TRANSITION_EVENT : "bao gồm"

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
        int network_latency_ms
        datetime last_frame_received_at
    }
    SYNC_BUFFER {
        string ingest_stream_id PK, FK
        int target_delay_ms
        int current_buffered_ms
    }
    SWITCH_COMMAND {
        string id PK
        string webinar_id FK
        string target_presenter_id FK
        datetime issued_at
        string issued_by
    }
    TRANSITION_EVENT {
        string id PK
        string switch_command_id FK
        string from_presenter_id FK
        string to_presenter_id FK
        string transition_type "cut | crossfade"
        datetime command_issued_at
        datetime applied_at
        int crossfade_duration_ms
    }
    DISCONNECT_EVENT {
        string id PK
        string presenter_id FK
        datetime detected_at
        string fallback_target "last_slide | other_presenter"
        datetime recovered_at
    }
    AUDIO_MIX_DECISION {
        string id PK
        string webinar_id FK
        string active_speaker_presenter_id FK
        string muted_presenter_ids
        datetime decided_at
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
    RECORDING {
        string id PK
        string webinar_id FK
        datetime started_at
        datetime ended_at
    }
```
