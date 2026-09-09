# Enhance ERD — Thêm device certificate, posture policy theo phòng ban và fallback access

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, `SESSION` giờ gắn với 1 `DEVICE` cụ thể thay vì chỉ gắn với `USER`, và có 4 entity mới, ứng trực tiếp với các yêu cầu:

- `DEVICE` và `DEVICE_CERTIFICATE` (mới) — đáp ứng yêu cầu 1 (request phải kèm certificate do MDM cấp, server verify còn hạn/chưa bị revoke/đúng chuỗi tin cậy trước khi cấp session) và yêu cầu 2 (thiết bị không có certificate hợp lệ, `mdm_enrolled=false`, bị từ chối với lý do rõ ràng).
- `POSTURE_POLICY` (mới), gắn với `DEPARTMENT` thay vì gắn cứng toàn công ty — đáp ứng yêu cầu 4 (chính sách OS tối thiểu, bắt buộc mã hóa... cấu hình được theo từng nhóm người dùng/phòng ban).
- `POSTURE_EVENT` (mới) ghi lại báo cáo posture theo thời gian thực từ MDM, dùng để phát hiện thiết bị chuyển sang không đạt chuẩn — đáp ứng yêu cầu 3 (buộc kết thúc session gần như ngay lập tức khi posture xấu đi).
- `FALLBACK_ACCESS_REQUEST` (mới) — đáp ứng yêu cầu 5 (luồng dự phòng VPN tạm thời + xác minh thủ công bởi IT, có log đầy đủ và giới hạn thời gian).

```mermaid
erDiagram
    DEPARTMENT ||--o{ USER : has
    DEPARTMENT ||--|| POSTURE_POLICY : "áp dụng"
    USER ||--o{ DEVICE : registers
    DEVICE ||--o{ DEVICE_CERTIFICATE : "được cấp"
    DEVICE ||--o{ POSTURE_EVENT : "báo cáo bởi MDM"
    USER ||--o{ SESSION : creates
    DEVICE ||--o{ SESSION : "dùng để truy cập"
    USER ||--o{ FALLBACK_ACCESS_REQUEST : requests

    DEPARTMENT {
        string id PK
        string name
    }
    USER {
        string id PK
        string email
        string department_id FK
        string password_hash
    }
    DEVICE {
        string id PK
        string user_id FK
        string device_name
        bool mdm_enrolled
        string os_version
        bool disk_encrypted
        bool jailbroken_or_rooted
    }
    DEVICE_CERTIFICATE {
        string id PK
        string device_id FK
        string serial_number
        string issued_by_mdm
        datetime valid_from
        datetime valid_to
        bool revoked
        string trust_chain_status "valid|broken"
    }
    POSTURE_POLICY {
        string id PK
        string department_id FK
        string min_os_version
        bool require_disk_encryption
        bool require_mdm_managed
        bool block_jailbroken_rooted
    }
    POSTURE_EVENT {
        string id PK
        string device_id FK
        string event_type "posture_ok|posture_degraded"
        string detail "vd disk_encryption_disabled|jailbreak_detected"
        datetime reported_at
    }
    SESSION {
        string id PK
        string user_id FK
        string device_id FK
        datetime created_at
        datetime expires_at
        string status "active|expired|revoked|force_terminated"
    }
    FALLBACK_ACCESS_REQUEST {
        string id PK
        string user_id FK
        string reason
        string vpn_grant_id
        string approved_by_it_user_id
        datetime requested_at
        datetime expires_at
        string status "pending|approved|expired|denied"
    }
```
