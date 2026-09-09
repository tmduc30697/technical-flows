# ERD - Base (trước khi có graceful shutdown cho streaming)

Đây là **base**: mô hình dữ liệu tối thiểu của một server phát video theo yêu cầu, phục vụ nhiều client qua kết nối HTTP dài. Chỉ giữ lại các entity cần để hiểu vì sao shutdown là vấn đề: một `Instance` đang giữ nhiều `StreamSession` sống lâu, mỗi session gắn với một `Video` có `duration_seconds` xác định (căn cứ để enhance sau này tính phân vị thời lượng và ước tính thời gian còn lại).

```mermaid
erDiagram
    CLIENT ||--o{ STREAM_SESSION : opens
    VIDEO ||--o{ STREAM_SESSION : is_played_in
    INSTANCE ||--o{ STREAM_SESSION : serves

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
        string status
        datetime started_at
        datetime ended_at
    }
```
