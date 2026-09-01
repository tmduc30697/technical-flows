# Sequence — Login with Email/Password

**Lớp: black-box** — **Nhóm: Hiện trạng**. Diagram này thể hiện thứ tự message của action "Login with email/password" đã liệt kê ở Use case baseline — mỗi action trong Use case ứng với đúng 1 sequence, và đây là action duy nhất của nhóm Hiện trạng.

```mermaid
sequenceDiagram
    actor U as Employee
    participant FE as Dashboard Frontend
    participant AUTH as Auth Service
    participant DB as User DB

    U->>FE: Nhập email + password, submit
    FE->>AUTH: POST /login {email, password}
    AUTH->>DB: SELECT user WHERE email = ?
    DB-->>AUTH: user row (password_hash)
    AUTH->>AUTH: verify(password, password_hash)
    alt password hợp lệ
        AUTH->>AUTH: issue session token (JWT/cookie)
        AUTH-->>FE: 200 OK {session token}
        FE-->>U: Redirect vào dashboard
    else password sai / user không tồn tại
        AUTH-->>FE: 401 Unauthorized
        FE-->>U: Hiện lỗi "Sai email hoặc mật khẩu"
    end
```
