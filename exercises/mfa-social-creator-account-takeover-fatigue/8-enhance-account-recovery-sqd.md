# Sequence Diagram — Enhance: Phục hồi tài khoản (tiered — ưu tiên cho creator lớn)

Đây là **enhance** của flow `account-recovery` ở base. So với base — nơi mọi người dùng đi chung một luồng email verification — enhance tách riêng một nhánh ưu tiên (`PRIORITY`) cho tài khoản `influence_tier=HIGH_VALUE`, với xác minh danh tính tăng cường và giám sát chống lạm dụng sau khôi phục, nhằm cân bằng giữa tốc độ khôi phục và rủi ro kẻ tấn công lợi dụng kênh ưu tiên.

```mermaid
sequenceDiagram
    actor User as Creator (mất thiết bị MFA)
    participant App as Client App
    participant Auth as Auth Service
    participant Support as Priority Support Team
    participant Standard as Standard Recovery Automation

    User->>App: Chọn "Không truy cập được MFA?"
    App->>Auth: POST /recovery/request {email}
    Auth->>Auth: Kiểm tra USER.influence_tier

    alt influence_tier = HIGH_VALUE
        Auth->>Auth: Tạo RECOVERY_REQUEST (tier=PRIORITY, status=PENDING)
        Auth->>Support: Escalate ticket kèm ngữ cảnh (follower_count, hoạt động gần đây)
        Support->>User: Liên hệ qua kênh xác minh trước đó (gọi số điện thoại đã đăng ký)
        User->>Support: Hoàn tất xác minh danh tính tăng cường (giấy tờ + video call)
        Support->>Auth: Duyệt khôi phục, đánh dấu reviewed_by
        Auth->>Auth: Cập nhật RECOVERY_REQUEST status=COMPLETED
        Auth->>Auth: Áp thêm giám sát rủi ro tạm thời sau khôi phục (hạn chế đăng bài/đổi thông tin 24-48h)
    else influence_tier = STANDARD
        Auth->>Auth: Tạo RECOVERY_REQUEST (tier=STANDARD, status=PENDING)
        Auth->>Standard: Gửi email verification link (giống luồng base)
        User->>App: Click link, xác thực lại password
        Standard->>Auth: Đánh dấu status=VERIFIED
        Auth->>Auth: Cập nhật RECOVERY_REQUEST status=COMPLETED
    end

    Auth->>Auth: Vô hiệu hoá MFA_METHOD cũ
    Auth-->>App: Khôi phục quyền truy cập, yêu cầu enroll lại MFA theo policy hiện hành
    App-->>User: Thông báo khôi phục thành công
```

**So với base:** thêm nhánh rẽ theo `influence_tier` — creator lớn được xử lý qua `Priority Support Team` với xác minh danh tính tăng cường và giám sát chống lạm dụng sau khôi phục, thay vì đi chung một luồng email-link như mọi tài khoản khác ở base.
