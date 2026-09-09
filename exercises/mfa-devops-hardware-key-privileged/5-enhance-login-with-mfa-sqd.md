# Enhance sequence — Login with MFA (chặn SMS OTP, bắt buộc WebAuthn, kiểm tra origin/RP ID)

Đây là **enhance** của flow `login-with-mfa` đã có ở base. So với base (chấp nhận mọi factor kể cả SMS OTP), nay tài khoản có `has_production_access` bị chặn hoàn toàn nếu cố dùng SMS OTP dù đã enroll trước đó, và phải xác thực bằng WebAuthn với origin/RP ID được kiểm tra khớp chuẩn. Đáp ứng yêu cầu 1 và yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor SRE as Kỹ sư SRE (có quyền production)
    participant Server
    participant DB as Database
    participant Browser as Browser (WebAuthn API)

    SRE->>Server: POST /login (email, password)
    Server->>DB: Xác thực password
    DB-->>Server: OK
    Server->>DB: SELECT ROLE WHERE user=SRE
    DB-->>Server: has_production_access=true

    SRE->>Server: Chọn xác thực bằng SMS OTP (factor cũ đã enroll)
    Server->>DB: SELECT MFA_FACTOR WHERE user=SRE, type=sms_otp
    DB-->>Server: allowed_for_production=false
    Server-->>SRE: Từ chối, "SMS OTP không hợp lệ cho tài khoản có quyền production, dùng security key"

    SRE->>Browser: Yêu cầu xác thực WebAuthn
    Browser->>Server: Gửi assertion (credential_id, origin, rp_id đo được từ browser)
    Server->>DB: SELECT SECURITY_KEY WHERE credential_id=... AND revoked_at IS NULL
    DB-->>Server: tìm thấy key, rp_id lưu = "console.internal.company.com"

    alt Origin/RP ID khớp với cấu hình
        Server->>Server: So khớp rp_id assertion với rp_id đã đăng ký, và origin với domain chính thức
        Server->>DB: Tạo LOGIN_SESSION (security_key_id=..., verified_rp_id=..., verified_origin=...)
        DB-->>Server: OK
        Server-->>SRE: Đăng nhập thành công
    else Origin/RP ID không khớp (vd domain giả mạo phishing)
        Server-->>SRE: Từ chối đăng nhập, "Origin không hợp lệ", không tiết lộ thêm chi tiết
    end
```
