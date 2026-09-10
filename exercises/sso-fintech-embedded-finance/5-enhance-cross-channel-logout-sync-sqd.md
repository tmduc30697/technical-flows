# Sequence Diagram — Enhance: Cross-Channel Logout Sync

Đây là **enhance**, flow hoàn toàn mới nối liền hai flow base "web-login" và "mobile-login": đăng xuất hoặc phát hiện bất thường trên một kênh giờ đây đẩy tín hiệu revoke qua push tới mọi kênh khác của cùng user gần như ngay lập tức, thay vì mỗi session tự sống độc lập tới khi hết hạn như ở base.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant Web as Web App
    participant Auth as Auth Service
    participant Mobile as Mobile App

    User->>Web: Bấm Đăng xuất trên web
    Web->>Auth: Yêu cầu revoke SESSION (channel=web)
    Auth->>Auth: Mark SESSION (channel=web) revoked
    Auth->>Auth: Tạo REVOCATION_EVENT cho user_id

    Auth->>Mobile: Push token revocation cho mọi SESSION khác của user
    Mobile->>Mobile: Nhận tín hiệu, hủy SESSION (channel=mobile) cục bộ ngay
    Mobile-->>User: Bắt buộc đăng nhập lại lần mở app tiếp theo

    Auth->>Auth: Ghi AUDIT_LOG cho từng session bị revoke
```
