# Enhance sequence — Enroll MFA factor (bắt buộc tối thiểu 2 security key trước khi cấp quyền production)

Đây là **enhance** của flow `enroll-mfa-factor` đã có ở base. So với base (chỉ cần 1 factor bất kỳ là được cấp quyền ngay), nay kỹ sư phải đăng ký ít nhất 2 security key (chính + dự phòng) trước khi tài khoản được cấp `production_access_granted_at`. Đáp ứng yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor SRE as Kỹ sư SRE
    participant Server
    participant DB as Database
    participant Browser as Browser (WebAuthn API)

    SRE->>Browser: Đăng ký security key thứ 1 (key chính)
    Browser->>Server: Gửi attestation (credential_id_1, public_key_1, rp_id, origin)
    Server->>DB: INSERT SECURITY_KEY (user=SRE, credential_id_1, label=chính)
    DB-->>Server: OK
    Server->>DB: SELECT COUNT(*) SECURITY_KEY WHERE user=SRE AND revoked_at IS NULL
    DB-->>Server: count=1
    Server-->>SRE: Đã đăng ký 1/2 key, cần thêm ít nhất 1 key dự phòng trước khi được cấp quyền production

    SRE->>Browser: Đăng ký security key thứ 2 (key dự phòng)
    Browser->>Server: Gửi attestation (credential_id_2, public_key_2, rp_id, origin)
    Server->>DB: INSERT SECURITY_KEY (user=SRE, credential_id_2, label=dự phòng)
    DB-->>Server: OK
    Server->>DB: SELECT COUNT(*) SECURITY_KEY WHERE user=SRE AND revoked_at IS NULL
    DB-->>Server: count=2

    Server->>DB: UPDATE USER SET production_access_granted_at=now() WHERE id=SRE
    DB-->>Server: OK
    Server-->>SRE: Đủ 2 key, tài khoản được cấp quyền truy cập production
```
