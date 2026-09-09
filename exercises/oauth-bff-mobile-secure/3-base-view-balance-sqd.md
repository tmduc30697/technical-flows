# Base sequence — View balance

Đây là **base**, flow "Xem số dư tài khoản" — chọn flow này vì đây chính là flow sẽ bị enhance tác động trực tiếp: ở base chỉ hiển thị số dư của 1 tài khoản duy nhất (tài khoản chính tại chính ngân hàng này), chưa có khái niệm tổng hợp số dư từ nhiều ngân hàng liên kết.

```mermaid
sequenceDiagram
    actor User
    participant MobileApp
    participant Backend as App Backend
    participant DB as Database

    User->>MobileApp: Mở màn hình số dư
    MobileApp->>Backend: GET /balance (session_token)
    Backend->>DB: Lấy BANK_ACCOUNT của user
    DB-->>Backend: balance
    Backend-->>MobileApp: balance
    MobileApp-->>User: Hiển thị số dư của 1 tài khoản duy nhất
    Note over MobileApp,Backend: Chưa có khái niệm liên kết ngân hàng đối tác nào khác
```
