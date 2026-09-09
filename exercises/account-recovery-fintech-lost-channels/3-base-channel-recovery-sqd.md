# Base sequence — Channel-based recovery (còn giữ được 1 kênh)

Đây là **base**, flow "Khôi phục qua OTP email/SMS" — quy trình khôi phục thông thường khi user **còn quyền truy cập ít nhất 1 kênh** đã đăng ký. Flow này liên quan mật thiết tới enhance vì nó chính là "đường dễ" sẽ **không áp dụng được** khi mất cả 2 kênh, buộc hệ thống phải chuyển sang quy trình xác minh danh tính thủ công/bán tự động mà đề bài yêu cầu.

```mermaid
sequenceDiagram
    actor User
    participant App as Fintech App
    participant DB as USER store
    participant Channel as Email/SMS Gateway

    User->>App: Chọn "Quên mật khẩu"
    App->>DB: Kiểm tra email/phone user nhập có khớp bản ghi
    DB-->>App: Khớp — còn kênh liên hệ hợp lệ
    App->>Channel: Gửi OTP tới email/phone đã đăng ký
    Channel-->>User: Nhận OTP
    User->>App: Nhập OTP
    App->>App: Xác minh OTP hợp lệ
    App-->>User: Cho phép đặt lại password ngay, không đóng băng gì
    Note over App,DB: Không có bước xác minh danh tính bổ sung, không audit trail riêng, không cooling-off
```
