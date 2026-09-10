# Sequence Diagram — Enhance: Clinic Staff SSO Scoped Access

Đây là **enhance**, flow hoàn toàn mới cho nhân viên phòng khám liên kết — mở rộng nguyên lý xác thực + ghi log của flow base "hospital-staff-view-record" nhưng thêm ràng buộc phạm vi theo `TREATMENT_RELATIONSHIP` và session ngắn hạn bắt buộc xác thực lại thường xuyên, phù hợp với máy trạm dùng chung nhiều ca kíp.

```mermaid
sequenceDiagram
    actor ClinicStaff as Nhân viên phòng khám
    participant Hospital as Bệnh viện trung tâm (IdP)
    participant EHR as Hệ thống EHR trung tâm

    ClinicStaff->>Hospital: Đăng nhập SSO (tài khoản phòng khám liên kết)
    Hospital->>Hospital: Xác thực, tạo SSO_SESSION thời gian sống ngắn
    Hospital-->>ClinicStaff: SSO thành công

    ClinicStaff->>EHR: Yêu cầu xem hồ sơ bệnh nhân X
    EHR->>EHR: Kiểm tra TREATMENT_RELATIONSHIP giữa clinic và patient X
    alt có quan hệ điều trị hợp lệ
        EHR->>EHR: Ghi ACCESS_LOG (clinic_staff_id, record_id, reason, accessed_at)
        EHR-->>ClinicStaff: Trả về hồ sơ bệnh nhân
    else không có quan hệ điều trị
        EHR-->>ClinicStaff: Từ chối, bệnh nhân không thuộc phạm vi điều trị của phòng khám
    end

    Note over Hospital,EHR: SSO_SESSION hết hạn nhanh hơn hệ thống nội bộ thông thường, bắt buộc xác thực lại thường xuyên hơn
```
