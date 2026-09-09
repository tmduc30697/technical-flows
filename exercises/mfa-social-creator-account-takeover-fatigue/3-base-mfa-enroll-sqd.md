# Sequence Diagram — Base: Đăng ký MFA (tự nguyện)

Đây là **base**, flow người dùng tự nguyện bật MFA (SMS hoặc Push) trong cài đặt bảo mật, không có ràng buộc nào dựa trên follower_count. Flow này được chọn làm nền so sánh vì enhance sẽ biến việc nâng cấp MFA từ "tự nguyện" thành "bắt buộc" đối với tài khoản ảnh hưởng lớn.

```mermaid
sequenceDiagram
    actor User as Creator
    participant App as Client App
    participant Auth as Auth Service

    User->>App: Vào Security Settings
    App->>Auth: GET /mfa/methods
    Auth-->>App: Trả về danh sách MFA hiện có (vd NONE)
    User->>App: Chọn "Bật SMS" hoặc "Bật Push"
    App->>Auth: POST /mfa/enroll {type}
    Auth->>User: Gửi mã xác thực (SMS OTP hoặc yêu cầu xác nhận trên thiết bị)
    User->>App: Nhập mã xác thực
    App->>Auth: Xác nhận mã
    Auth->>Auth: Tạo MFA_METHOD (is_active=true)
    Auth-->>App: Enroll thành công
    App-->>User: Thông báo đã bật MFA
```

**Đặc điểm base:** hoàn toàn tự chọn, không có ngưỡng follower nào kích hoạt, không có deadline/grace period.
