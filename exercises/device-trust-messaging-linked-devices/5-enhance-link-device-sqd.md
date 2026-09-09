# Enhance sequence — Liên kết thiết bị mới qua QR code, phải duyệt từ thiết bị chính

Đây là **enhance**, flow hoàn toàn mới thay thế cho việc "đăng nhập thiết bị mới sẽ đăng xuất thiết bị cũ" ở base. Thiết bị web/desktop muốn liên kết phải hiển thị QR code, và chỉ trở thành thiết bị liên kết thật sự sau khi thiết bị chính quét và xác nhận — không cho phép liên kết chỉ bằng cách nhập lại thông tin đăng nhập từ xa. Đáp ứng yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant Web as Web (muốn liên kết)
    participant Phone as Điện thoại (thiết bị chính)
    participant Server
    participant DB as Database

    User->>Web: Mở web.messaging.app, không nhập username/password
    Web->>Server: Yêu cầu tạo LINK_REQUEST mới
    Server->>DB: INSERT LINK_REQUEST (qr_token, requesting_device_type=web, status=pending)
    Server-->>Web: Trả về QR code chứa qr_token

    User->>Phone: Mở app, chọn "Liên kết thiết bị", quét QR code trên màn hình web
    Phone->>Server: Gửi qr_token vừa quét kèm session_id của thiết bị chính
    Server->>DB: Kiểm tra LINK_REQUEST còn hiệu lực (chưa hết hạn, chưa dùng)
    DB-->>Server: Hợp lệ

    Phone-->>User: Hiển thị xác nhận "Cho phép liên kết thiết bị Web này?"
    User->>Phone: Xác nhận đồng ý

    Phone->>Server: Gửi approve LINK_REQUEST
    Server->>DB: UPDATE LINK_REQUEST SET status=approved, approved_by_session_id=Phone.session_id
    Server->>DB: INSERT DEVICE_SESSION (role=linked, device_type=web, encryption_key_id=khoá riêng mới sinh, status=active)
    Server->>DB: UPDATE LINK_REQUEST SET resulting_session_id=session vừa tạo

    Server-->>Web: Liên kết thành công, nhận session riêng cùng khoá mã hoá riêng
    Web-->>User: Web hiện đã đăng nhập, đồng thời điện thoại vẫn hoạt động bình thường

    Note over Phone,Web: Nếu không có thao tác quét/duyệt từ điện thoại trong thời gian quy định, LINK_REQUEST tự động hết hạn, không có cách nào liên kết chỉ bằng thông tin đăng nhập từ xa
```
