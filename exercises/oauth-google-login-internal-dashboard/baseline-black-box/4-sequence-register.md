# Sequence — Register (Email/Password)

**Lớp: black-box** — **Nhóm: Hiện trạng**. Diagram này thể hiện action "Register (email/password)" đã bổ sung vào Use case baseline (`1-usecase-baseline-auth.md`) — tiền đề bắt buộc để có account cho action Login xác thực vào.

```mermaid
sequenceDiagram
    actor U as Employee
    participant FE as Dashboard Frontend
    participant AUTH as Auth Service
    participant DB as User DB

    U->>FE: Nhập email + password + confirm password
    FE->>AUTH: POST /register {email, password}
    AUTH->>DB: SELECT user WHERE email = ?
    alt email đã tồn tại
        DB-->>AUTH: user row
        AUTH-->>FE: 409 Conflict "email đã được đăng ký"
        FE-->>U: Hiện lỗi
    else email chưa tồn tại
        DB-->>AUTH: không có row
        AUTH->>AUTH: hash password
        AUTH->>DB: INSERT user {email, password_hash}
        AUTH->>AUTH: issue session token (JWT/cookie) — auto-login sau khi register
        AUTH-->>FE: 201 Created {session token}
        FE-->>U: Redirect vào dashboard
    end
```

**Ghi chú thiết kế:** Register tự động đăng nhập luôn sau khi tạo account thành công (dùng chung `Session Issuer` với Login, xem `baseline-white-box/1-component-auth-service.md`) — tránh bắt user phải login lại lần nữa ngay sau khi vừa đăng ký.
