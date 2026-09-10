# ERD — Base (trước khi có index thời gian thực)

Đây là **base**: mô hình dữ liệu suy luận cho mạng xã hội *trước khi* có pipeline index gần thời gian thực. Đề bài giả định đã có `USER`, `POST` (có thể chứa `HASHTAG` trích xuất từ nội dung), và quan hệ `FOLLOW` giữa người dùng — đây là dữ liệu nền bắt buộc để "tìm kiếm người dùng/hashtag/bài viết kèm cá nhân hóa theo follow" có nghĩa. Tìm kiếm ở base được suy luận là chạy trên index đồng bộ theo lô định kỳ, chưa đáp ứng được yêu cầu gần thời gian thực.

```mermaid
erDiagram
    USER ||--o{ POST : creates
    USER ||--o{ FOLLOW : "follows (as follower)"
    POST ||--o{ POST_HASHTAG : tags
    HASHTAG ||--o{ POST_HASHTAG : "used in"

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
```
