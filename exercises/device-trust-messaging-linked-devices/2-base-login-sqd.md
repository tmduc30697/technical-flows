# Base sequence — Login (đăng nhập thiết bị mới sẽ đăng xuất thiết bị cũ)

Đây là **base**, flow đăng nhập ở trạng thái hiện tại: app chỉ cho phép 1 session hoạt động, đăng nhập thành công trên thiết bị mới sẽ tự động đăng xuất session đang hoạt động trên thiết bị cũ. Đây là nền để so sánh với yêu cầu 1 của đề bài — thay vì đăng xuất thiết bị cũ, hệ thống cần cho phép thêm thiết bị liên kết song song với thiết bị chính, nhưng phải qua xác nhận trực tiếp từ thiết bị chính.

```mermaid
sequenceDiagram
    actor User
    participant Phone as Điện thoại (session cũ)
    participant Web as Web (thiết bị mới)
    participant Server

    Note over Phone,Server: Điện thoại đang có session active

    User->>Web: Đăng nhập bằng số điện thoại + mã OTP
    Web->>Server: Xác thực OTP
    Server-->>Web: Xác thực thành công

    Server->>Phone: Thu hồi session cũ (revoke)
    Server->>Web: INSERT SESSION mới (device_type=web, status=active)

    Phone-->>User: Bị đăng xuất khỏi điện thoại ngay lập tức
    Web-->>User: Đăng nhập thành công trên web

    Note over Phone,Web: Không thể dùng đồng thời cả 2 thiết bị, chỉ 1 session được active tại một thời điểm
```
