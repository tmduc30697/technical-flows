# Base sequence — Check following status (đọc cùng R thấp, có thể sai nút bấm)

Đây là **base**, flow "Kiểm tra đã follow chưa để hiển thị nút Follow/Following" — dùng cùng read_quorum thấp như đọc counter, có thể đọc phải replica trễ. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 2 của đề bài chính là sửa đúng lỗ hổng này.

```mermaid
sequenceDiagram
    actor User
    participant App as Social App
    participant Config as QUORUM_CONFIG store
    participant Rel as RELATIONSHIP_REPLICA (node bất kỳ)

    User->>App: Vừa bấm Follow xong (API trả thành công)
    App->>App: Load lại trang, cần hiển thị đúng nút Follow/Following
    App->>Config: Lấy read_quorum chung (thấp, giống đọc counter)
    App->>Rel: Đọc status từ node bất kỳ
    Rel-->>App: Trả status=not_following (replica chưa kịp cập nhật)
    App-->>User: Hiển thị nhầm nút "Follow" dù đã follow thành công
    Note over App,Rel: User bấm Follow lần nữa vì tưởng chưa follow — có thể gây ghi trùng/logic sai ở nơi khác dựa vào trạng thái này
```
