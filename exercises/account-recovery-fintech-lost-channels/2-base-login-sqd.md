# Base sequence — Login

Đây là **base**, flow "Đăng nhập bằng email/phone + password" — tiền đề trực tiếp cho enhance: chính vì tài khoản gắn với 2 kênh liên hệ cố định (email, phone) để xác thực/khôi phục, nên khi mất cả 2 kênh này, các cơ chế thông thường (login lẫn recovery chuẩn) đều vô hiệu — đó là case khó mà đề bài yêu cầu xử lý.

```mermaid
sequenceDiagram
    actor User
    participant App as Fintech App
    participant DB as USER store

    User->>App: Nhập email/phone + password
    App->>DB: Tra password_hash theo email/phone
    DB-->>App: Trả về password_hash
    App->>App: So khớp hash
    App->>App: Tạo SESSION mới
    App-->>User: Đăng nhập thành công, truy cập ví/tài khoản đầu tư
```
