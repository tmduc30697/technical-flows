# Enhance ERD — WebAuthn/FIDO2 bắt buộc, tối thiểu 2 key, revoke riêng lẻ, cảnh báo bất thường

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, thay `MFA_FACTOR` chung chung bằng entity chuyên biệt `SECURITY_KEY` cho WebAuthn, và thêm các entity/trường sau, ứng trực tiếp với từng yêu cầu:

- `MFA_FACTOR.allowed_for_production` (false với `sms_otp`) — đáp ứng yêu cầu 1 (chặn hoàn toàn SMS OTP cho tài khoản production).
- `SECURITY_KEY` (mới, thay thế phần WebAuthn của `MFA_FACTOR`) với `credential_id`, `public_key`, `rp_id`, `origin`, `revoked_at` — đáp ứng yêu cầu 4 (kiểm tra origin/RP ID chuẩn WebAuthn) và yêu cầu 3 (revoke từng key riêng lẻ qua `revoked_at`, không xóa các key khác).
- `USER.production_access_granted_at` chỉ được set khi đủ điều kiện — đáp ứng yêu cầu 2 (bắt buộc tối thiểu 2 security key trước khi cấp quyền).
- `SECURITY_ALERT` (mới) — đáp ứng yêu cầu 5 (cảnh báo/khóa tạm thời khi phát hiện nhiều lần enroll liên tiếp trong thời gian ngắn).

```mermaid
erDiagram
    USER ||--|| ROLE : "được gán"
    USER ||--o{ MFA_FACTOR : enroll
    USER ||--o{ SECURITY_KEY : "đăng ký"
    USER ||--o{ LOGIN_SESSION : "đăng nhập"
    USER ||--o{ SECURITY_ALERT : "phát sinh cảnh báo"

    USER {
        string id PK
        string email
        datetime production_access_granted_at "null nếu chưa đủ điều kiện"
    }
    ROLE {
        string id PK
        string name
        boolean has_production_access
    }
    MFA_FACTOR {
        string id PK
        string user_id FK
        string type "sms_otp|totp"
        boolean allowed_for_production "false cho sms_otp"
        datetime enrolled_at
    }
    SECURITY_KEY {
        string id PK
        string user_id FK
        string credential_id
        string public_key
        string rp_id "relying party ID kỳ vọng"
        string origin "origin đã dùng lúc đăng ký"
        string label "vd: chính, dự phòng"
        datetime created_at
        datetime revoked_at "null nếu còn hiệu lực, set khi revoke riêng lẻ"
    }
    LOGIN_SESSION {
        string id PK
        string user_id FK
        string security_key_id FK
        string verified_rp_id
        string verified_origin
        datetime created_at
    }
    SECURITY_ALERT {
        string id PK
        string user_id FK
        string type "rapid_enrollment"
        datetime triggered_at
        datetime account_locked_until
    }
```
