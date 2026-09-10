# Sequence Diagram — Base: Mobile Login

Đây là **base**, flow đăng nhập trên mobile app, hoàn toàn độc lập với session web — cùng với flow web-login, đây là tiền đề bắt buộc để enhance xây cơ chế đồng bộ trạng thái đăng nhập xuyên kênh.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant Mobile as Mobile App
    participant Auth as Auth Service

    User->>Mobile: Đăng nhập (email/password)
    Mobile->>Auth: Xác thực
    Auth->>Auth: Tạo SESSION (channel=mobile)
    Auth-->>Mobile: session_token
    Mobile-->>User: Đăng nhập thành công
    Note over Mobile,Auth: Session này độc lập hoàn toàn, không liên quan gì tới session web hiện có
```
