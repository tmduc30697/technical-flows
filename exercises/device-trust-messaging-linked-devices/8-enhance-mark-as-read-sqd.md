# Enhance sequence — Mark as read (2 thiết bị liên kết cùng thao tác gần như đồng thời)

Đây là **enhance** của flow `mark-as-read` đã có ở base. So với base (chỉ 1 session nên không có tranh chấp), nay nhiều thiết bị liên kết có thể cùng đánh dấu đã đọc gần như đồng thời. Thay vì so sánh `updated_at` (dễ sai lệch do đồng hồ hệ thống khác nhau giữa các thiết bị), hệ thống dùng `last_read_message_seq` tăng dần đơn điệu theo hội thoại và chỉ merge theo giá trị lớn nhất, đảm bảo nhất quán dù request nào tới server trước. Đáp ứng yêu cầu 5 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant Phone as Điện thoại (main)
    participant Web as Web (linked)
    participant Server
    participant DB as Database

    Note over Phone,Web: User đang mở cùng 1 hội thoại trên cả điện thoại lẫn web, cả 2 gần như cùng lúc đọc tới cuối hội thoại

    par Gần như đồng thời
        Phone->>Server: Đánh dấu đã đọc tới message seq=15
    and
        Web->>Server: Đánh dấu đã đọc tới message seq=14 (do web tải chậm hơn 1 nhịp)
    end

    Server->>DB: UPDATE READ_STATE SET last_read_message_seq = GREATEST(last_read_message_seq, 15) WHERE conversation_id=... AND user_id=...
    DB-->>Server: Request từ Phone áp dụng thành công, last_read_message_seq=15

    Server->>DB: UPDATE READ_STATE SET last_read_message_seq = GREATEST(last_read_message_seq, 14) WHERE conversation_id=... AND user_id=...
    DB-->>Server: Request từ Web tới sau nhưng seq=14 nhỏ hơn giá trị hiện tại 15, GIỮ NGUYÊN 15, không bị ghi đè lùi lại

    Server-->>Phone: Đồng bộ trạng thái đã đọc = seq 15
    Server-->>Web: Đồng bộ trạng thái đã đọc = seq 15 (khớp với Phone dù request của Web tới sau và có seq nhỏ hơn)

    Note over Server,DB: Dùng GREATEST theo seq tăng dần đơn điệu thay vì ghi đè theo thứ tự request tới, nên dù 2 thiết bị gửi gần như đồng thời và tới server không theo đúng thứ tự thời gian thực, trạng thái cuối cùng luôn hội tụ về giá trị đã đọc xa nhất, không bao giờ bị lùi lại
```
