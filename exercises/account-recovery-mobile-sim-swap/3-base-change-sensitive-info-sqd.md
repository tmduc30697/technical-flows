# Base sequence — Change sensitive info (chỉ dựa vào OTP)

Đây là **base**, flow "Đổi email liên kết / phương thức khôi phục" ở trạng thái hiện tại — chỉ cần verify lại OTP là đủ. Đây chính là lỗ hổng mà toàn bộ 5 yêu cầu của đề bài nhắm vào: kẻ tấn công SIM-swap có OTP hợp lệ sẽ thực hiện được y hệt chủ tài khoản thật.

```mermaid
sequenceDiagram
    actor User
    participant App as Mobile App
    participant DB as USER / OTP_CODE store
    participant SMS as SMS Gateway

    User->>App: Yêu cầu đổi email liên kết (hoặc đổi phương thức khôi phục)
    App->>DB: Tạo OTP_CODE mới, gửi lại qua SMS để "xác nhận"
    App->>SMS: Gửi OTP
    SMS-->>User: Nhận OTP
    User->>App: Nhập OTP
    App->>DB: Xác minh OTP khớp
    DB-->>App: Hợp lệ
    App->>DB: Áp dụng thay đổi ngay lập tức
    App-->>User: Đổi thành công
    Note over App,DB: Không có factor độc lập nào khác ngoài OTP, không cảnh báo thiết bị tin cậy khác, không cooling-off nếu số vừa đổi SIM
```
