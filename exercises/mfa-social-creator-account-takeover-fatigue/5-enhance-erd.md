# ERD — Enhance (sau khi áp dụng đề bài)

Đây là **enhance**: mô hình dữ liệu sau khi áp toàn bộ yêu cầu — nâng cấp MFA tự động theo ngưỡng ảnh hưởng, chống MFA fatigue, và phục hồi khẩn cấp phân tier. So với base, các entity mới/thay đổi: `USER` thêm `influence_tier`; `MFA_METHOD` mở rộng type (`TOTP`/`WEBAUTHN`) và `is_required`; thêm `MFA_POLICY_RULE` + `MFA_UPGRADE_TASK` (ngưỡng follower → bắt buộc nâng cấp); `MFA_CHALLENGE` thêm ngữ cảnh (`device_info`, `location`, `display_code` cho number matching); thêm `PUSH_RATE_LIMIT_WINDOW` (giới hạn tần suất push); thêm `SECURITY_LOCK_EVENT` + `SECURITY_ALERT` (phát hiện tấn công → khoá nguồn + cảnh báo kênh độc lập); `RECOVERY_REQUEST` thêm `tier` và `reviewed_by` (phục hồi ưu tiên cho creator lớn).

```mermaid
erDiagram
    USER {
        uuid user_id PK
        string username
        string email
        string password_hash
        int follower_count
        string influence_tier "STANDARD or HIGH_VALUE"
        datetime created_at
    }
    FOLLOW {
        uuid follow_id PK
        uuid follower_id FK
        uuid followee_id FK
        datetime created_at
    }
    MFA_METHOD {
        uuid mfa_method_id PK
        uuid user_id FK
        string type "NONE or SMS or PUSH or TOTP or WEBAUTHN"
        string phone_number
        boolean is_active
        boolean is_required
        datetime enrolled_at
    }
    MFA_POLICY_RULE {
        uuid rule_id PK
        int min_follower_count
        string required_mfa_type
        int grace_period_days
    }
    MFA_UPGRADE_TASK {
        uuid task_id PK
        uuid user_id FK
        uuid triggered_by_rule_id FK
        string required_mfa_type
        string status "PENDING or COMPLETED or EXPIRED"
        datetime triggered_at
        datetime grace_deadline_at
    }
    LOGIN_ATTEMPT {
        uuid attempt_id PK
        uuid user_id FK
        string ip_address
        string device_info
        string location
        string status
        datetime created_at
    }
    MFA_CHALLENGE {
        uuid challenge_id PK
        uuid attempt_id FK
        uuid user_id FK
        string status "PENDING or APPROVED or DENIED or EXPIRED"
        string device_info
        string location
        string display_code
        datetime created_at
        datetime responded_at
    }
    PUSH_RATE_LIMIT_WINDOW {
        uuid window_id PK
        uuid user_id FK
        datetime window_start
        int push_count
        datetime locked_until
    }
    SECURITY_LOCK_EVENT {
        uuid event_id PK
        uuid user_id FK
        string source_ip
        int consecutive_denial_count
        datetime locked_at
        datetime unlocked_at
    }
    SECURITY_ALERT {
        uuid alert_id PK
        uuid event_id FK
        string channel "EMAIL or SMS or BACKUP_CONTACT"
        string status
        datetime sent_at
    }
    RECOVERY_REQUEST {
        uuid request_id PK
        uuid user_id FK
        string tier "STANDARD or PRIORITY"
        string verification_method
        string status "PENDING or VERIFIED or COMPLETED or REJECTED"
        string reviewed_by
        datetime submitted_at
        datetime resolved_at
    }

    USER ||--o{ FOLLOW : follows
    USER ||--o{ FOLLOW : followed_by
    USER ||--o{ MFA_METHOD : has
    USER ||--o{ MFA_UPGRADE_TASK : must_complete
    MFA_POLICY_RULE ||--o{ MFA_UPGRADE_TASK : evaluated_by
    USER ||--o{ LOGIN_ATTEMPT : initiates
    LOGIN_ATTEMPT ||--o| MFA_CHALLENGE : triggers
    USER ||--o{ MFA_CHALLENGE : must_approve
    USER ||--o{ PUSH_RATE_LIMIT_WINDOW : rate_limited_by
    USER ||--o{ SECURITY_LOCK_EVENT : triggers
    SECURITY_LOCK_EVENT ||--o{ SECURITY_ALERT : generates
    USER ||--o{ RECOVERY_REQUEST : submits
```
