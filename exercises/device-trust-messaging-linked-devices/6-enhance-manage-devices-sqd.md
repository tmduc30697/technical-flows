# Enhance sequence — Xem danh sách thiết bị liên kết, revoke từng thiết bị riêng lẻ

Đây là **enhance**, flow hoàn toàn mới, chưa tồn tại ở base (base không có khái niệm nhiều thiết bị để quản lý). Người dùng xem đầy đủ danh sách thiết bị liên kết đang hoạt động ngay trên thiết bị chính, và revoke được từng thiết bị riêng lẻ ngay lập tức mà không ảnh hưởng tới thiết bị chính hay các thiết bị liên kết khác, nhờ mỗi thiết bị có khoá mã hoá session riêng. Đáp ứng yêu cầu 2 và 3 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant Phone as Điện thoại (thiết bị chính)
    participant Server
    participant DB as Database
    participant Web1 as Web (thiết bị liên kết 1)
    participant Desktop as Desktop (thiết bị liên kết 2)

    User->>Phone: Mở màn hình "Thiết bị đã liên kết"
    Phone->>Server: Lấy danh sách DEVICE_SESSION của user
    Server->>DB: SELECT DEVICE_SESSION WHERE user_id=... AND status=active
    DB-->>Server: Danh sách gồm Phone (main), Web1 (linked, linked_at, last_active_at), Desktop (linked, linked_at, last_active_at)
    Server-->>Phone: Hiển thị đầy đủ danh sách kèm thời điểm liên kết và lần hoạt động cuối

    User->>Phone: Chọn revoke Web1
    Phone->>Server: Yêu cầu revoke DEVICE_SESSION(Web1)
    Server->>DB: UPDATE DEVICE_SESSION(Web1) SET status=revoked
    Server->>DB: Xoá encryption_key_id riêng của Web1

    Server-->>Web1: Đẩy thông báo real-time buộc đăng xuất ngay lập tức
    Web1-->>User: Web1 tự động đăng xuất

    Note over Desktop,Phone: Desktop và Phone không hề bị ảnh hưởng, vẫn hoạt động bình thường vì mỗi thiết bị dùng khoá mã hoá session độc lập
    Phone-->>User: Danh sách thiết bị cập nhật, không còn Web1
```
