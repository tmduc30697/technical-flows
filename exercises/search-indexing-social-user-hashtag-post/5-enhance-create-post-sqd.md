# Sequence Diagram — Enhance: Create Post

Đây là **enhance**, flow "đăng bài viết" đã tồn tại ở base ([2-base-create-post-sqd.md](2-base-create-post-sqd.md)) nay thay đổi cốt lõi: thay vì chờ batch sync, bài viết phát `INDEX_SYNC_EVENT` được xử lý gần thời gian thực, và pipeline phải chịu được khối lượng ghi hàng nghìn bài/giây mà không làm chậm độ trễ đọc của các truy vấn tìm kiếm khác đang chạy song song.

```mermaid
sequenceDiagram
    actor User
    participant PostSvc as Post Service
    participant DB as Posts + Hashtags DB
    participant Queue as Index Sync Event Queue (write lane)
    participant Indexer as Index Sync Worker Pool
    participant Index as Search Index (read replicas)

    User->>PostSvc: Đăng bài viết mới (giờ cao điểm, hàng nghìn bài/giây)
    PostSvc->>DB: INSERT INTO posts, trích xuất hashtag liên quan
    DB-->>PostSvc: post_id created
    PostSvc->>Queue: Publish INDEX_SYNC_EVENT (post_id, hashtag_ids)
    PostSvc-->>User: Đăng bài thành công

    Queue->>Indexer: Nhiều worker xử lý song song, tách riêng lane ghi khỏi lane đọc
    Indexer->>Index: Ghi POST_INDEX_DOCUMENT vào index, qua write path riêng
    Index-->>Indexer: Indexed gần thời gian thực

    Note over Queue,Index: Write path tách khỏi read replicas phục vụ truy vấn, nên khối lượng ghi lớn không làm chậm độ trễ đọc của người dùng khác
```
