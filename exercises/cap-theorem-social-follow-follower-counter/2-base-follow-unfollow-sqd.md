# Base sequence — Follow/unfollow (ghi trực tiếp, không version)

Đây là **base**, flow "Follow/unfollow" ở trạng thái hiện tại — ghi quan hệ và cập nhật counter trực tiếp, dùng cùng quorum cố định, không có version/timestamp để xử lý thứ tự. Flow này liên quan mật thiết tới enhance vì yêu cầu 2 và 3 của đề bài chính là sửa đúng lỗ hổng ở đây.

```mermaid
sequenceDiagram
    actor User
    participant App as Social App
    participant Config as QUORUM_CONFIG store
    participant Rel as FOLLOW_RELATIONSHIP store
    participant Counter as FOLLOWER_COUNT store

    User->>App: Bấm Follow
    App->>Config: Lấy write_quorum chung
    App->>Rel: Ghi status=following (không có version/timestamp)
    Rel-->>App: Ghi thành công
    App->>Counter: Tăng count += 1
    Counter-->>App: Cập nhật thành công
    App-->>User: "Đã follow"

    Note over App,Rel: Nếu user bấm Unfollow ngay sau đó (double-tap do UI lag), request unfollow có thể tới 1 node khác gần như đồng thời, không có cách nào xác định request nào thực sự là ý định cuối cùng
```
