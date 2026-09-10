# ERD — Base (trước khi có SSO cho mạng lưới phòng khám liên kết)

Đây là **base**: mô hình dữ liệu suy luận cho hệ thống bệnh viện trung tâm *trước khi* mở SSO cho phòng khám liên kết. Đề bài giả định bệnh viện trung tâm đã có sẵn hệ thống hồ sơ bệnh án (EHR) với nhân viên y tế nội bộ và access log cơ bản khi xem hồ sơ — nếu chưa có STAFF/PATIENT/MEDICAL_RECORD thì việc "cho phép phòng khám liên kết SSO vào" sẽ không có nghĩa. Base chưa có khái niệm CLINIC, quan hệ điều trị, hay audit chi tiết theo từng lượt xem.

```mermaid
erDiagram
    STAFF ||--o{ ACCESS_LOG : "views records via"
    PATIENT ||--o{ MEDICAL_RECORD : owns
    MEDICAL_RECORD ||--o{ ACCESS_LOG : "is target of"

    STAFF {
        string staff_id PK
        string full_name
        string role
        string password_hash
    }

    PATIENT {
        string patient_id PK
        string full_name
        date date_of_birth
    }

    MEDICAL_RECORD {
        string record_id PK
        string patient_id FK
        string content_summary
        datetime updated_at
    }

    ACCESS_LOG {
        string log_id PK
        string staff_id FK
        string record_id FK
        datetime accessed_at
    }
```
