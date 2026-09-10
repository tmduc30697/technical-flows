# ERD — Enhance (sau khi có SSO cho mạng lưới phòng khám liên kết)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng yêu cầu đề bài — phòng khám liên kết và nhân viên của họ, quan hệ điều trị giới hạn phạm vi xem hồ sơ, session SSO ngắn hạn cho nhân viên y tế, audit log chi tiết theo từng lượt xem, hợp đồng phòng khám để revoke đồng loạt, và luồng truy cập khẩn cấp có kiểm soát. So với base, `ACCESS_LOG` nay bắt buộc gắn với danh tính nhân viên cụ thể và lý do truy cập, còn quyền xem hồ sơ không còn "SSO xong là xem được tất cả" mà bị giới hạn bởi `TREATMENT_RELATIONSHIP`.

```mermaid
erDiagram
    CLINIC ||--o{ CLINIC_STAFF : employs
    CLINIC ||--o{ CLINIC_CONTRACT : "bound by"
    CLINIC_STAFF ||--o{ SSO_SESSION : "logs in via"
    CLINIC_STAFF ||--o{ TREATMENT_RELATIONSHIP : "treats via clinic"
    PATIENT ||--o{ TREATMENT_RELATIONSHIP : "treated by"
    PATIENT ||--o{ MEDICAL_RECORD : owns
    CLINIC_STAFF ||--o{ ACCESS_LOG : "views records via"
    MEDICAL_RECORD ||--o{ ACCESS_LOG : "is target of"
    CLINIC_STAFF ||--o{ EMERGENCY_ACCESS_GRANT : "may request"

    CLINIC {
        string clinic_id PK
        string name
    }

    CLINIC_CONTRACT {
        string contract_id PK
        string clinic_id FK
        string status
        date terminated_at
    }

    CLINIC_STAFF {
        string clinic_staff_id PK
        string clinic_id FK
        string full_name
        string role
    }

    SSO_SESSION {
        string session_id PK
        string clinic_staff_id FK
        datetime issued_at
        datetime expires_at
        datetime revoked_at
        boolean reauth_required
    }

    PATIENT {
        string patient_id PK
        string full_name
        date date_of_birth
    }

    TREATMENT_RELATIONSHIP {
        string relationship_id PK
        string patient_id FK
        string clinic_id FK
        string status
        date started_at
    }

    MEDICAL_RECORD {
        string record_id PK
        string patient_id FK
        string content_summary
        datetime updated_at
    }

    ACCESS_LOG {
        string log_id PK
        string clinic_staff_id FK
        string record_id FK
        string reason
        datetime accessed_at
    }

    EMERGENCY_ACCESS_GRANT {
        string grant_id PK
        string clinic_staff_id FK
        string patient_id FK
        string reason
        datetime granted_at
        datetime expires_at
        string reviewed_by
    }
```
