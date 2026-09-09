# Base sequence — Login với MFA cố định

Đây là **base**, flow "Đăng nhập" ở trạng thái hiện tại — sau khi đúng mật khẩu, hệ thống luôn yêu cầu OTP với mọi vai trò, mọi lần, không phân biệt rủi ro. Flow này liên quan mật thiết tới enhance vì toàn bộ yêu cầu của đề bài là làm cho bước "yêu cầu MFA hay không" trở nên thích ứng theo rủi ro thay vì cố định như ở đây.

```mermaid
sequenceDiagram
    actor User as Người dùng (bệnh nhân hoặc bác sĩ)
    participant App as Auth Service
    participant DB as LOGIN_SESSION / OTP_CODE store

    User->>App: Nhập email + mật khẩu
    App->>App: Kiểm tra password_hash
    App->>DB: Tạo LOGIN_SESSION(mfa_verified=false)
    Note over App: MFA luôn bắt buộc, không phân biệt vai trò hay tín hiệu rủi ro
    App->>DB: Sinh OTP_CODE cho session
    App-->>User: Yêu cầu nhập OTP
    User->>App: Nhập OTP
    App->>DB: Kiểm tra OTP_CODE hợp lệ và chưa dùng
    App->>DB: Cập nhật LOGIN_SESSION(mfa_verified=true)
    App-->>User: Đăng nhập thành công, cho truy cập hồ sơ bệnh án
```
