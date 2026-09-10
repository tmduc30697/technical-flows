# Sequence Diagram — Base: Web Login

Đây là **base**, flow đăng nhập trên web app, tạo session độc lập với mobile — tiền đề để so sánh với enhance, nơi đăng xuất trên web phải phản ánh gần như ngay lập tức trên mobile qua token revocation thay vì hai kênh tách biệt hoàn toàn.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant Web as Web App
    participant Auth as Auth Service

    User->>Web: Đăng nhập (email/password)
    Web->>Auth: Xác thực
    Auth->>Auth: Tạo SESSION (channel=web)
    Auth-->>Web: session_token
    Web-->>User: Đăng nhập thành công
```
