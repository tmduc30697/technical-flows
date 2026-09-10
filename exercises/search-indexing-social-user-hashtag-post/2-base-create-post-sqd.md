# Sequence Diagram — Base: Create Post

Đây là **base**, flow "đăng bài viết" khi index chỉ đồng bộ theo lô định kỳ — tiền đề cho thấy hạn chế: bài mới không xuất hiện ngay trong tìm kiếm, không phù hợp cho sự kiện đang nóng cần gần thời gian thực.

```mermaid
sequenceDiagram
    actor User
    participant PostSvc as Post Service
    participant DB as Posts + Hashtags DB
    participant BatchSync as Periodic Sync Job (batch)
    participant Index as Search Index

    User->>PostSvc: Đăng bài viết mới (có thể chứa hashtag)
    PostSvc->>DB: INSERT INTO posts, trích xuất và INSERT/UPDATE hashtags liên quan
    DB-->>PostSvc: post_id created
    PostSvc-->>User: Đăng bài thành công

    Note over DB,Index: Bài viết chưa xuất hiện trong tìm kiếm ngay
    BatchSync->>DB: Quét thay đổi theo chu kỳ định kỳ
    BatchSync->>Index: Đồng bộ bài viết/hashtag mới vào index
    Index-->>BatchSync: Indexed

    Note over BatchSync,Index: Có độ trễ đáng kể, không phù hợp cho sự kiện breaking news/viral
```
