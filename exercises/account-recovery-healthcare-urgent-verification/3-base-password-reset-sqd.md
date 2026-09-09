# Base sequence — Password reset (mức xác minh phổ biến)

Đây là **base**, flow "Quên mật khẩu" ở mức xác minh phổ biến — chỉ dựa vào việc sở hữu email, giống chuẩn thường thấy ở đa số hệ thống. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 4 của đề bài chính là nâng cấp mức xác minh của flow này lên cao hơn do dữ liệu sức khỏe nhạy cảm hơn.

```mermaid
sequenceDiagram
    actor Patient
    participant App as Healthcare Platform
    participant DB as PATIENT store
    participant Email as Email Gateway

    Patient->>App: Chọn "Quên mật khẩu", nhập email
    App->>DB: Kiểm tra email có tồn tại
    DB-->>App: Khớp
    App->>Email: Gửi link reset password
    Email-->>Patient: Nhận link qua email
    Patient->>App: Mở link, đặt mật khẩu mới
    App->>DB: Cập nhật password_hash
    App-->>Patient: Đổi mật khẩu thành công, đăng nhập lại
    Note over App,DB: Chỉ dựa vào quyền sở hữu email, không có bước xác minh danh tính bổ sung
```
