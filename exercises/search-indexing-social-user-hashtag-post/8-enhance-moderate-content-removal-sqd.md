# Sequence Diagram — Enhance: Moderate Content Removal

Đây là **enhance**, flow hoàn toàn mới: khi bài viết/tài khoản bị xóa hoặc khóa do vi phạm chính sách, nội dung phải biến mất khỏi kết quả tìm kiếm và autocomplete ngay lập tức, đồng bộ với hành động kiểm duyệt.

```mermaid
sequenceDiagram
    actor Moderator as Đội kiểm duyệt
    participant ModSvc as Moderation Service
    participant DB as Posts/Users DB
    participant Queue as Index Sync Event Queue (priority cao)
    participant Indexer as Index Sync Worker
    participant Index as Search Index
    actor Searcher as Người tìm kiếm

    Moderator->>ModSvc: Xóa bài viết hoặc khóa tài khoản vi phạm
    ModSvc->>DB: UPDATE posts/users SET status = removed/banned
    DB-->>ModSvc: Updated
    ModSvc->>Queue: Publish INDEX_SYNC_EVENT (change_type = moderation_removal), priority cao nhất

    Queue->>Indexer: Deliver ngay lập tức, ưu tiên trước các event thường
    Indexer->>Index: Xóa/ẩn document khỏi search index và autocomplete
    Index-->>Indexer: Removed

    ModSvc-->>Moderator: Đã xử lý kiểm duyệt
    Searcher->>Index: Tìm kiếm nội dung/tài khoản đã bị xử lý
    Index-->>Searcher: Không còn xuất hiện, không có khoảng trễ để nội dung vi phạm vẫn tìm được
```
