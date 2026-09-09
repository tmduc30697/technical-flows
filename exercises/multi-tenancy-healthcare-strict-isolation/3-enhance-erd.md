# Enhance ERD — sau khi có cách ly nghiêm ngặt theo tenant

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 5 nhóm thay đổi/entity mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `TENANT` (sửa) — thêm `isolation_model` (database riêng hoặc schema riêng theo tenant, không còn chỉ cột `tenant_id` trong shared schema).
- `TENANT_ENCRYPTION_KEY` (mới, thay cho `ENCRYPTION_KEY` dùng chung) — mỗi tenant một khóa riêng, hỗ trợ crypto shredding khi cần huỷ dữ liệu một tenant mà không ảnh hưởng tenant khác.
- `TENANT_DELETION_REQUEST` + `DELETION_TASK` (mới) — theo dõi tiến trình xóa dữ liệu triệt để trên từng hệ thống lưu trữ (DB chính, cache, backup, log), có xác nhận hoàn tất trong khung thời gian xác định.
- `AUDIT_LOG` (sửa) — vẫn gắn `tenant_id` như base nhưng nay được lưu/truy vấn tách biệt theo tenant, không admin hệ thống thông thường nào truy vấn xuyên tenant được.
- `ANONYMIZED_PATIENT_RECORD` + `ANALYTICS_MODEL` (mới) — dữ liệu đã ẩn danh hóa dùng để huấn luyện mô hình phân tích xu hướng, tách khỏi dữ liệu định danh gốc.

```mermaid
erDiagram
    TENANT ||--o{ STAFF_USER : employs
    TENANT ||--o{ PATIENT : "has"
    PATIENT ||--o{ MEDICAL_RECORD : "has"
    TENANT ||--|| TENANT_ENCRYPTION_KEY : "encrypted with (riêng biệt mỗi tenant)"
    MEDICAL_RECORD }o--|| TENANT_ENCRYPTION_KEY : "encrypted at rest by"
    STAFF_USER ||--o{ AUDIT_LOG : generates
    MEDICAL_RECORD ||--o{ AUDIT_LOG : "accessed via"
    TENANT ||--o{ AUDIT_LOG : "isolated per tenant"
    TENANT ||--o{ TENANT_DELETION_REQUEST : requests
    TENANT_DELETION_REQUEST ||--o{ DELETION_TASK : spawns
    PATIENT ||--o{ ANONYMIZED_PATIENT_RECORD : "anonymized into"
    ANALYTICS_MODEL ||--o{ ANONYMIZED_PATIENT_RECORD : "trained from"

    TENANT {
        string id PK
        string name
        string isolation_model "dedicated_db | dedicated_schema"
        datetime created_at
    }
    STAFF_USER {
        string id PK
        string tenant_id FK
        string email
        string role "doctor | staff | system_admin"
        string password_hash
    }
    PATIENT {
        string id PK
        string tenant_id FK
        string name
        date dob
    }
    MEDICAL_RECORD {
        string id PK
        string tenant_id FK
        string patient_id FK
        string content
        datetime updated_at
    }
    TENANT_ENCRYPTION_KEY {
        string id PK
        string tenant_id FK
        string key_material
        string status "active | revoked"
        datetime created_at
        datetime revoked_at "set khi crypto shredding"
    }
    AUDIT_LOG {
        string id PK
        string tenant_id FK
        string staff_user_id FK
        string medical_record_id FK
        string action "view | edit"
        datetime accessed_at
        string storage_partition "riêng theo tenant, không có API xuyên tenant"
    }
    TENANT_DELETION_REQUEST {
        string id PK
        string tenant_id FK
        string requested_by
        datetime requested_at
        datetime deadline_at
        string status "pending | in_progress | completed"
        datetime completed_at
        string confirmation_receipt
    }
    DELETION_TASK {
        string id PK
        string deletion_request_id FK
        string target_store "primary_db | cache | backup | log"
        string status "pending | done | failed"
        datetime completed_at
    }
    ANONYMIZED_PATIENT_RECORD {
        string id PK
        string tenant_id FK
        string source_patient_id FK "chỉ để trace nội bộ, không xuất ra ngoài"
        string anonymization_method "vd k-anonymity, generalization"
        datetime created_at
    }
    ANALYTICS_MODEL {
        string id PK
        string version
        datetime trained_at
        string training_scope "chỉ trên dữ liệu đã anonymize, gộp nhiều tenant"
    }
```
