# Base sequence — Login with MFA (mọi phương thức được chấp nhận như nhau, kể cả SMS OTP)

Đây là **base**, flow đăng nhập hiện tại: hệ thống chấp nhận bất kỳ `MFA_FACTOR` nào user đã enroll, kể cả SMS OTP, dù tài khoản có quyền truy cập production hay không. Đây là tiền đề cho yêu cầu 1 của đề bài — hiện tại chưa có sự phân biệt độ mạnh giữa các phương thức MFA.

```mermaid
sequenceDiagram
    actor SRE as Kỹ sư SRE (có quyền production)
    participant Server
    participant DB as Database

    SRE->>Server: POST /login (email, password)
    Server->>DB: Xác thực password
    DB-->>Server: OK
    Server->>DB: SELECT MFA_FACTOR WHERE user_id=SRE
    DB-->>Server: 1 factor, type=sms_otp

    Server-->>SRE: Gửi mã OTP qua SMS
    SRE->>Server: Nhập mã OTP đúng
    Server->>DB: Tạo LOGIN_SESSION (mfa_factor_used=sms_otp)
    DB-->>Server: OK
    Server-->>SRE: Đăng nhập thành công, được cấp toàn quyền production

    Note over Server,DB: Hệ thống không hề kiểm tra "SMS OTP có đủ mạnh cho tài khoản production hay không", chỉ cần có 1 factor hợp lệ là qua
```
