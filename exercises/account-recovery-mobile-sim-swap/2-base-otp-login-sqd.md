# Base sequence — OTP Login

Đây là **base**, flow "Đăng nhập bằng OTP" — nền tảng cho enhance: chính vì hệ thống coi việc nhận đúng OTP qua SMS là bằng chứng danh tính, nên kẻ tấn công chiếm được số điện thoại (SIM-swap) cũng vượt qua được bước này y hệt chủ tài khoản thật.

```mermaid
sequenceDiagram
    actor User
    participant App as Mobile App
    participant DB as USER / OTP_CODE store
    participant SMS as SMS Gateway

    User->>App: Nhập số điện thoại
    App->>DB: Tạo OTP_CODE mới
    App->>SMS: Gửi OTP tới số điện thoại
    SMS-->>User: Nhận OTP
    User->>App: Nhập OTP
    App->>DB: Xác minh OTP khớp, chưa hết hạn
    DB-->>App: Hợp lệ
    App->>DB: Đăng ký/cập nhật DEVICE hiện tại, tạo SESSION
    App-->>User: Đăng nhập thành công
```
