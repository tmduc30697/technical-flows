# Base ERD — SaaS đa phòng khám trước khi siết cách ly nghiêm ngặt

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** áp các lớp bảo vệ nghiêm ngặt. Đề bài đối chiếu trực tiếp với "chỉ cột `tenant_id` trong shared schema" (yêu cầu 1) — nên base được suy luận là một SaaS đa tenant kiểu shared schema thông thường: mọi phòng khám (tenant) chia sẻ cùng database/schema, phân biệt dữ liệu bằng cột `tenant_id`, dùng **một khóa mã hóa dùng chung cho toàn platform** (chưa tách theo tenant — đối lập với yêu cầu 2), đã có audit log cơ bản khi xem hồ sơ bệnh nhân nhưng **chưa được cách ly/giới hạn xem theo tenant** (đối lập với yêu cầu 4, admin hệ thống có thể xem log của mọi tenant). Base **chưa có** quy trình xóa dữ liệu triệt để theo tenant (yêu cầu 3) và **chưa có** pipeline AI/phân tích ẩn danh hóa (yêu cầu 5) — những phần này là enhance hoàn toàn mới.

```mermaid
erDiagram
    TENANT ||--o{ STAFF_USER : employs
    TENANT ||--o{ PATIENT : "has"
    PATIENT ||--o{ MEDICAL_RECORD : "has"
    ENCRYPTION_KEY ||--o{ MEDICAL_RECORD : "encrypts (1 khóa dùng chung toàn platform)"
    STAFF_USER ||--o{ AUDIT_LOG : generates
    MEDICAL_RECORD ||--o{ AUDIT_LOG : "accessed via"

    TENANT {
        string id PK
        string name
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
    ENCRYPTION_KEY {
        string id PK
        string key_material
        string scope "platform-wide (không tách theo tenant)"
    }
    AUDIT_LOG {
        string id PK
        string tenant_id FK
        string staff_user_id FK
        string medical_record_id FK
        string action "view | edit"
        datetime accessed_at
    }
```
