# Enhance ERD — sau khi có phân tầng khẩn cấp/nâng xác minh/ủy quyền giám hộ

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 3 nhóm entity mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `IDENTITY_VERIFICATION` (mới) — mức xác minh nâng cao (không chỉ dựa email) cho khôi phục thông thường.
- `EMERGENCY_ACCESS_GRANT` + `EMERGENCY_ACCESS_LOG` + `POST_INCIDENT_REVIEW` (mới) — luồng khẩn cấp riêng cho nhân viên y tế: phạm vi giới hạn (`scope`), tự hết hạn (`expires_at`), log chi tiết từng field đã xem, và review sau sự việc tự động kích hoạt.
- `GUARDIAN_RELATIONSHIP` + `GUARDIAN_RECOVERY_REQUEST` (mới) — quy trình ủy quyền cho người thân/giám hộ hợp pháp, tách biệt hẳn khỏi flow tự khôi phục của chính bệnh nhân.
- `MEDICAL_STAFF` (mới) — actor yêu cầu quyền khẩn cấp và thực hiện review.

```mermaid
erDiagram
    PATIENT ||--|| MEDICAL_RECORD : has
    PATIENT ||--|| BILLING_INFO : has
    PATIENT ||--o{ SESSION : creates
    PATIENT ||--o{ IDENTITY_VERIFICATION : "verified via"
    PATIENT ||--o{ EMERGENCY_ACCESS_GRANT : "granted on"
    MEDICAL_STAFF ||--o{ EMERGENCY_ACCESS_GRANT : requests
    EMERGENCY_ACCESS_GRANT ||--o{ EMERGENCY_ACCESS_LOG : records
    EMERGENCY_ACCESS_GRANT ||--o| POST_INCIDENT_REVIEW : "triggers"
    MEDICAL_STAFF ||--o{ POST_INCIDENT_REVIEW : reviews
    PATIENT ||--o{ GUARDIAN_RELATIONSHIP : "has guardian"
    GUARDIAN_RELATIONSHIP ||--o{ GUARDIAN_RECOVERY_REQUEST : "initiates"

    PATIENT {
        string id PK
        string email
        string password_hash
        string name
        date dob
    }
    MEDICAL_RECORD {
        string id PK
        string patient_id FK
        string allergies
        string current_medications
        string diagnosis_notes
        string full_history
    }
    BILLING_INFO {
        string id PK
        string patient_id FK
        string payment_method
        string insurance_details
    }
    SESSION {
        string id PK
        string patient_id FK
        datetime created_at
        datetime expires_at
    }
    MEDICAL_STAFF {
        string id PK
        string name
        string department
        string role
    }
    IDENTITY_VERIFICATION {
        string id PK
        string patient_id FK
        string method "id_document | multi_factor_kba | video_verification"
        string status
        datetime verified_at
    }
    EMERGENCY_ACCESS_GRANT {
        string id PK
        string patient_id FK
        string requested_by_staff_id FK
        string justification
        string scope "allergies, current_medications"
        string status
        datetime granted_at
        datetime expires_at
    }
    EMERGENCY_ACCESS_LOG {
        string id PK
        string grant_id FK
        string field_viewed
        datetime viewed_at
    }
    POST_INCIDENT_REVIEW {
        string id PK
        string grant_id FK
        string reviewer_staff_id FK
        string review_outcome
        datetime reviewed_at
    }
    GUARDIAN_RELATIONSHIP {
        string id PK
        string patient_id FK
        string guardian_user_id
        string relationship_type
        string legal_proof_ref
        string verification_status
        datetime verified_at
    }
    GUARDIAN_RECOVERY_REQUEST {
        string id PK
        string guardian_relationship_id FK
        string status
        datetime created_at
        datetime resolved_at
    }
```
