# Base sequence — Shop owner login

Đây là **base**, flow "Chủ shop đăng nhập vào trang quản trị" — tiền đề bắt buộc cho enhance, vì màn hình consent (nơi user quyết định approve quyền cho app thứ ba) chỉ có nghĩa khi user đã đăng nhập và xác định được đây là user nào, quản lý shop nào.

```mermaid
sequenceDiagram
    actor Owner as Shop Owner
    participant AdminDashboard
    participant AuthServer
    participant DB as Database

    Owner->>AdminDashboard: Nhập email/password
    AdminDashboard->>AuthServer: POST /login
    AuthServer->>DB: Xác thực user, xác định shop quản lý
    DB-->>AuthServer: user hợp lệ, shop_id
    AuthServer->>DB: Tạo session
    DB-->>AuthServer: session_token
    AuthServer-->>AdminDashboard: session_token
    AdminDashboard-->>Owner: Vào trang quản trị shop
```
