# Enhance ERD — sau khi có phòng vệ SIM-swap

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm entity mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `SENSITIVE_ACTION_REQUEST` (mới) — mọi yêu cầu đổi email/phương thức khôi phục đều phải đi qua request có theo dõi second factor riêng, thay vì chỉ verify OTP như base.
- `TRUSTED_DEVICE_ALERT` (mới) — cảnh báo tới thiết bị tin cậy trước khi thực thi thay đổi đến từ số điện thoại vừa nhận OTP mới.
- `SIM_SWAP_EVENT` (mới) — ghi nhận tín hiệu đổi SIM (từ nhà mạng hoặc suy luận từ thiết bị lạ), tính risk_level, áp cooling-off.
- `PRIORITY_RECOVERY_CASE` + `ROLLED_BACK_ACTION` (mới) — luồng khôi phục ưu tiên không phụ thuộc số điện thoại, khóa/hoàn tác các thay đổi kẻ tấn công đã thực hiện.

```mermaid
erDiagram
    USER ||--o{ DEVICE : uses
    USER ||--o{ SESSION : creates
    USER ||--o{ OTP_CODE : requests
    DEVICE ||--o{ SESSION : hosts
    USER ||--o{ SENSITIVE_ACTION_REQUEST : initiates
    SENSITIVE_ACTION_REQUEST ||--o| TRUSTED_DEVICE_ALERT : triggers
    DEVICE ||--o{ TRUSTED_DEVICE_ALERT : "receives (as trusted device)"
    USER ||--o{ SIM_SWAP_EVENT : "flagged with"
    USER ||--o{ PRIORITY_RECOVERY_CASE : opens
    PRIORITY_RECOVERY_CASE ||--o{ ROLLED_BACK_ACTION : reverts
    SENSITIVE_ACTION_REQUEST ||--o| ROLLED_BACK_ACTION : "may be target of"

    USER {
        string id PK
        string phone
        string email
        string backup_email
        string recovery_method
    }
    DEVICE {
        string id PK
        string user_id FK
        string device_fingerprint
        boolean trusted
        datetime first_seen_at
        datetime last_seen_at
    }
    SESSION {
        string id PK
        string user_id FK
        string device_id FK
        datetime created_at
        datetime expires_at
    }
    OTP_CODE {
        string id PK
        string user_id FK
        string phone
        string code
        datetime created_at
        datetime expires_at
        datetime verified_at
    }
    SENSITIVE_ACTION_REQUEST {
        string id PK
        string user_id FK
        string action_type "change_email | change_recovery_method"
        string phone_otp_status
        string second_factor_status
        string status "pending | blocked | completed"
        datetime created_at
        datetime executed_at
    }
    TRUSTED_DEVICE_ALERT {
        string id PK
        string sensitive_action_request_id FK
        string trusted_device_id FK
        datetime sent_at
        string response "approved | blocked | no_response"
        datetime responded_at
    }
    SIM_SWAP_EVENT {
        string id PK
        string user_id FK
        datetime detected_at
        string signal_source "carrier_signal | new_device_otp"
        string risk_level "low | high"
        datetime cooling_off_until
        string status
    }
    PRIORITY_RECOVERY_CASE {
        string id PK
        string user_id FK
        string reason "reported_sim_swap"
        string verification_method "backup_email | manual_id_verification"
        string status
        datetime created_at
        datetime resolved_at
    }
    ROLLED_BACK_ACTION {
        string id PK
        string priority_recovery_case_id FK
        string sensitive_action_request_id FK
        datetime rolled_back_at
    }
```
