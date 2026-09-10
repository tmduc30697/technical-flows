# Sequence Diagram — Base: Hospital Staff View Record

Đây là **base**, flow nhân viên y tế nội bộ bệnh viện đăng nhập và xem hồ sơ bệnh án với access log cơ bản — tiền đề bắt buộc vì enhance sẽ mở rộng đúng cơ chế xác thực và ghi log này cho nhân viên phòng khám liên kết, nhưng có thêm ràng buộc phạm vi và chi tiết hơn.

```mermaid
sequenceDiagram
    actor Staff as Nhân viên y tế bệnh viện
    participant EHR as Hệ thống EHR trung tâm

    Staff->>EHR: Đăng nhập (email/password nội bộ)
    EHR->>EHR: Xác thực, tạo session
    Staff->>EHR: Yêu cầu xem hồ sơ bệnh nhân
    EHR->>EHR: Truy vấn MEDICAL_RECORD
    EHR->>EHR: Ghi ACCESS_LOG (staff_id, record_id, accessed_at)
    EHR-->>Staff: Trả về hồ sơ bệnh nhân
```
