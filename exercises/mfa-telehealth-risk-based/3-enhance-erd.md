# Enhance ERD — sau khi có adaptive MFA theo rủi ro

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 5 nhóm thay đổi/entity mới, ứng trực tiếp với các yêu cầu trong đề bài:

- `LOGIN_SESSION` (sửa) — thêm `device_id`, `ip_address`, `geo_country`, `geo_city` để ghi nhận tín hiệu ngữ cảnh của lần đăng nhập.
- `KNOWN_DEVICE` (mới) — thiết bị đã từng đăng nhập thành công của một user, dùng để so sánh "thiết bị đã biết hay chưa".
- `RISK_ASSESSMENT` (mới) — kết quả tính điểm rủi ro cho một session: thiết bị/vị trí đã biết chưa, có bất thường giờ truy cập không, điểm số và mức rủi ro, kèm cờ báo lỗi/timeout của dịch vụ tính rủi ro để phục vụ fail-safe.
- `MFA_DECISION_LOG` (mới) — log bắt buộc mọi quyết định yêu cầu/miễn MFA kèm lý do, phục vụ audit compliance.
- `ESCALATED_VERIFICATION` (mới) — yêu cầu xác minh mạnh hơn (liên hệ hỗ trợ) khi rủi ro cao bất thường, thay vì chỉ hỏi lại OTP.

```mermaid
erDiagram
    USER ||--o{ LOGIN_SESSION : "logs in"
    USER ||--o{ KNOWN_DEVICE : "has trusted"
    KNOWN_DEVICE ||--o{ LOGIN_SESSION : "seen in"
    LOGIN_SESSION ||--o| RISK_ASSESSMENT : "scored by"
    LOGIN_SESSION ||--o{ OTP_CODE : "may generate"
    LOGIN_SESSION ||--o{ MFA_DECISION_LOG : "logged as"
    LOGIN_SESSION ||--o| ESCALATED_VERIFICATION : "may trigger"
    USER ||--o{ MEDICAL_RECORD : "owns (as patient)"

    USER {
        string id PK
        string email
        string password_hash
        string role "patient | doctor"
        datetime created_at
    }
    KNOWN_DEVICE {
        string id PK
        string user_id FK
        string device_fingerprint
        datetime first_seen_at
        datetime last_seen_at
    }
    LOGIN_SESSION {
        string id PK
        string user_id FK
        string device_id FK "nullable, thiết bị chưa từng thấy thì null"
        string ip_address
        string geo_country
        string geo_city
        datetime started_at
        boolean mfa_required
        boolean mfa_verified
    }
    RISK_ASSESSMENT {
        string id PK
        string session_id FK
        boolean device_known
        boolean location_known
        boolean time_anomaly
        int risk_score
        string risk_level "low | high | critical"
        boolean risk_service_error "true nếu dịch vụ chấm điểm lỗi/timeout"
        datetime created_at
    }
    OTP_CODE {
        string id PK
        string session_id FK
        string code_hash
        datetime expires_at
        boolean used
    }
    MFA_DECISION_LOG {
        string id PK
        string session_id FK
        string user_id FK
        boolean mfa_required
        string decision_reason "vd device_unknown | doctor_role | risk_service_timeout"
        string risk_level
        datetime created_at
    }
    ESCALATED_VERIFICATION {
        string id PK
        string session_id FK
        string user_id FK
        string method "contact_support"
        string status "pending | resolved"
        datetime created_at
    }
    MEDICAL_RECORD {
        string id PK
        string patient_id FK
        string content
        datetime updated_at
    }
```
