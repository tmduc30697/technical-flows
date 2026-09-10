# ERD — Enhance (sau khi có index gần thời gian thực)

Đây là **enhance**: thêm `HASHTAG_TREND_METRIC` tính theo cửa sổ thời gian (growth rate) thay vì tổng lượt dùng lịch sử, `MODERATION_ACTION` đồng bộ việc gỡ nội dung vi phạm vào index ngay lập tức, và `INDEX_SYNC_EVENT` thay cho batch sync để đạt độ trễ gần thời gian thực. So với base, `POST_INDEX_DOCUMENT`/`USER_INDEX_DOCUMENT` được cập nhật qua event-driven pipeline chịu tải ghi lớn, không còn phụ thuộc chu kỳ batch.

```mermaid
erDiagram
    USER ||--o{ POST : creates
    USER ||--o{ FOLLOW : "follows (as follower)"
    POST ||--o{ POST_HASHTAG : tags
    HASHTAG ||--o{ POST_HASHTAG : "used in"
    HASHTAG ||--o{ HASHTAG_TREND_METRIC : "tracked over time windows"
    POST ||--o{ INDEX_SYNC_EVENT : triggers
    USER ||--o{ INDEX_SYNC_EVENT : triggers
    INDEX_SYNC_EVENT ||--o| POST_INDEX_DOCUMENT : updates
    MODERATION_ACTION ||--o| INDEX_SYNC_EVENT : "also triggers removal"

    USER {
        string user_id PK
        string username
        string status
    }

    FOLLOW {
        string follow_id PK
        string follower_user_id FK
        string followed_user_id FK
    }

    POST {
        string post_id PK
        string user_id FK
        string content
        string status
        datetime created_at
    }

    HASHTAG {
        string hashtag_id PK
        string tag_text
        int total_usage_count
    }

    POST_HASHTAG {
        string post_id FK
        string hashtag_id FK
    }

    HASHTAG_TREND_METRIC {
        string metric_id PK
        string hashtag_id FK
        datetime window_start
        int usage_in_window
        decimal growth_rate
    }

    POST_INDEX_DOCUMENT {
        string post_id PK
        string content
        string user_id
        string status
        datetime indexed_at
    }

    USER_INDEX_DOCUMENT {
        string user_id PK
        string username
        int follower_count
    }

    MODERATION_ACTION {
        string action_id PK
        string target_type
        string target_id
        string action_type
        datetime acted_at
    }

    INDEX_SYNC_EVENT {
        string event_id PK
        string target_type
        string target_id
        string change_type
        string status
        datetime created_at
    }
```
