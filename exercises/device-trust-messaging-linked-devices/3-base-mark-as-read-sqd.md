# Base sequence — Mark as read (chỉ 1 session nên không có tranh chấp cập nhật)

Đây là **base**, flow đánh dấu đã đọc ở trạng thái hiện tại: vì chỉ có duy nhất 1 session hoạt động tại một thời điểm, việc cập nhật `READ_STATE` luôn tuần tự, không có tranh chấp ghi đồng thời. Đây là nền để so sánh với yêu cầu 5 của đề bài, vốn chỉ phát sinh khi cho phép nhiều thiết bị liên kết cùng hoạt động song song.

```mermaid
sequenceDiagram
    actor User
    participant Phone as Điện thoại (session duy nhất)
    participant Server
    participant DB as Database

    User->>Phone: Mở hội thoại, đọc tới tin nhắn mới nhất
    Phone->>Server: Đánh dấu đã đọc tới message_id=M10
    Server->>DB: UPDATE READ_STATE SET last_read_message_id=M10, updated_at=now()
    DB-->>Server: Cập nhật thành công
    Server-->>Phone: Xác nhận đã đồng bộ

    Note over Phone,DB: Vì không có thiết bị nào khác đang hoạt động song song, không bao giờ xảy ra việc 2 nơi cùng ghi READ_STATE cùng lúc
```
