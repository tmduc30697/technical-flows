# Sequence Diagram — Enhance: Đăng ký MFA (bắt buộc nâng cấp theo ngưỡng ảnh hưởng)

Đây là **enhance** của flow `mfa-enroll` ở base. So với base, việc nâng cấp MFA không còn hoàn toàn tự nguyện: khi follower_count vượt ngưỡng trong `MFA_POLICY_RULE`, hệ thống tự động tạo yêu cầu bắt buộc chuyển sang WebAuthn/TOTP, có thông báo trước và grace period để không đột ngột khoá quyền truy cập của creator đang hoạt động bình thường.

```mermaid
sequenceDiagram
    participant Job as Follower Growth Monitor
    participant Auth as Auth Service
    participant Notify as Notification Service
    actor User as Creator
    participant App as Client App

    Job->>Auth: USER.follower_count vừa vượt MFA_POLICY_RULE.min_follower_count
    Auth->>Auth: Cập nhật USER.influence_tier=HIGH_VALUE
    Auth->>Auth: Tạo MFA_UPGRADE_TASK (status=PENDING, required_mfa_type=WEBAUTHN, grace_deadline_at=now+N ngày)
    Auth->>Notify: Yêu cầu gửi thông báo bắt buộc nâng cấp MFA
    Notify-->>User: Email + in-app banner, "Cần thiết lập WebAuthn/TOTP trước hạn X"

    Note over User,App: Trong grace period, creator vẫn đăng nhập bình thường bằng MFA cũ

    User->>App: Vào Security Settings, chọn "Thiết lập WebAuthn/TOTP"
    App->>Auth: POST /mfa/enroll {type=WEBAUTHN}
    Auth->>User: Yêu cầu xác thực thiết bị (WebAuthn attestation hoặc quét mã TOTP)
    User->>App: Hoàn tất xác thực
    App->>Auth: Xác nhận enroll
    Auth->>Auth: Tạo MFA_METHOD mới (is_active=true, is_required=true)
    Auth->>Auth: Vô hiệu hoá MFA_METHOD cũ (SMS hoặc PUSH-only)
    Auth->>Auth: Cập nhật MFA_UPGRADE_TASK status=COMPLETED
    Auth-->>App: Nâng cấp MFA thành công

    opt Hết grace_deadline_at mà chưa hoàn tất
        Auth->>Auth: MFA_UPGRADE_TASK status=EXPIRED
        Auth->>Auth: Áp thêm bước xác minh khi login thay vì khoá hẳn tài khoản
    end
```

**So với base:** trigger không còn là hành động tự nguyện của user mà là một job hệ thống (`Follower Growth Monitor`) phát hiện ngưỡng ảnh hưởng bị vượt, kèm `grace_deadline_at` và nhánh xử lý khi hết hạn — thay vì chỉ có 1 luồng enroll đơn giản.
