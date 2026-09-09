# Sequence Diagram — Base: Phục hồi tài khoản khi mất MFA

Đây là **base**, flow phục hồi tài khoản chuẩn khi người dùng mất thiết bị MFA — duy nhất một đường xử lý, áp dụng như nhau cho mọi tài khoản bất kể follower_count. Flow này được chọn làm nền vì enhance sẽ tách riêng một nhánh phục hồi ưu tiên dành cho creator lớn.

```mermaid
sequenceDiagram
    actor User as Creator (mất thiết bị MFA)
    participant App as Client App
    participant Auth as Auth Service
    participant Email as Email Service

    User->>App: Chọn "Không truy cập được MFA?"
    App->>Auth: POST /recovery/request {email}
    Auth->>Auth: Tạo RECOVERY_REQUEST (verification_method=EMAIL_LINK, status=PENDING)
    Auth->>Email: Gửi link xác thực
    Email-->>User: Nhận email chứa link
    User->>App: Click link, nhập lại password
    App->>Auth: Xác thực link + password
    Auth->>Auth: Cập nhật RECOVERY_REQUEST status=VERIFIED
    Auth->>Auth: Vô hiệu hoá MFA_METHOD cũ
    Auth->>Auth: RECOVERY_REQUEST status=COMPLETED
    Auth-->>App: Khôi phục quyền truy cập, yêu cầu enroll lại MFA
    App-->>User: Thông báo khôi phục thành công
```

**Đặc điểm base:** một quy trình duy nhất cho tất cả người dùng, không có bước xác minh danh tính tăng cường, không có kênh hỗ trợ ưu tiên riêng.
