# ERD — Base (trước khi áp enhance)

Đây là **base**: mô hình dữ liệu mạng xã hội trước khi áp dụng chính sách MFA nâng cao cho creator lớn. Chỉ suy luận phần liên quan mật thiết tới enhance: người dùng, quan hệ follow (để biết mức độ ảnh hưởng), phương thức MFA hiện có (SMS/push đơn giản, chưa có WebAuthn/TOTP bắt buộc), lịch sử đăng nhập, challenge MFA, và một luồng phục hồi tài khoản chuẩn (chưa phân tier ưu tiên). Không suy diễn thêm các module không liên quan (đăng bài, billing, nhắn tin...).

```mermaid
erDiagram
    USER {
        uuid user_id PK
        string username
        string email
        string password_hash
        int follower_count
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
        string type "NONE or SMS or PUSH"
        string phone_number
        boolean is_active
        datetime enrolled_at
    }
    LOGIN_ATTEMPT {
        uuid attempt_id PK
        uuid user_id FK
        string ip_address
        string device_info
        string status
        datetime created_at
    }
    MFA_CHALLENGE {
        uuid challenge_id PK
        uuid attempt_id FK
        uuid user_id FK
        string status "PENDING or APPROVED or DENIED or EXPIRED"
        datetime created_at
        datetime responded_at
    }
    RECOVERY_REQUEST {
        uuid request_id PK
        uuid user_id FK
        string verification_method "EMAIL_LINK or SUPPORT_TICKET"
        string status "PENDING or VERIFIED or COMPLETED or REJECTED"
        datetime submitted_at
        datetime resolved_at
    }

    USER ||--o{ FOLLOW : follows
    USER ||--o{ FOLLOW : followed_by
    USER ||--o{ MFA_METHOD : has
    USER ||--o{ LOGIN_ATTEMPT : initiates
    LOGIN_ATTEMPT ||--o| MFA_CHALLENGE : triggers
    USER ||--o{ MFA_CHALLENGE : must_approve
    USER ||--o{ RECOVERY_REQUEST : submits
```

**Ghi chú suy luận:**
- `USER.follower_count` và `FOLLOW` phải tồn tại trước — enhance cần biết "tài khoản có ảnh hưởng lớn" để áp chính sách MFA mạnh hơn.
- `MFA_METHOD.type` ở base chỉ gồm `NONE`, `SMS`, `PUSH` — chưa có `WEBAUTHN`/`TOTP`, vì enhance yêu cầu "buộc chuyển từ SMS hoặc không có MFA sang WebAuthn/TOTP".
- Đã có `PUSH` như một lựa chọn MFA vì enhance mô tả rõ "thay vì chỉ một nút Approve đơn giản" — tức UI push approve đơn giản đã tồn tại trước, enhance chỉ nâng cấp nó.
- `RECOVERY_REQUEST` ở base chỉ có 1 luồng duy nhất, không phân biệt tier — enhance thêm phân tier ưu tiên cho creator lớn.
