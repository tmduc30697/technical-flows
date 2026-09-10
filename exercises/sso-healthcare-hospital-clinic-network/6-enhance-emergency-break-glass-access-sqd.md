# Sequence Diagram — Enhance: Emergency Break-Glass Access

Đây là **enhance**, flow hoàn toàn mới cho tình huống khẩn cấp lâm sàng (ví dụ bệnh nhân chuyển viện gấp) — cấp quyền truy cập mở rộng có kiểm soát và giới hạn thời gian, tách biệt khỏi luồng SSO thông thường bị giới hạn bởi `TREATMENT_RELATIONSHIP`, và luôn kèm log để review sau.

```mermaid
sequenceDiagram
    actor ClinicStaff as Nhân viên phòng khám
    participant EHR as Hệ thống EHR trung tâm
    actor Reviewer as Người review sau sự cố

    ClinicStaff->>EHR: Yêu cầu xem hồ sơ bệnh nhân X (không có TREATMENT_RELATIONSHIP)
    EHR-->>ClinicStaff: Từ chối theo luồng SSO thông thường

    ClinicStaff->>EHR: Yêu cầu Emergency Access, nêu lý do khẩn cấp lâm sàng
    EHR->>EHR: Tạo EMERGENCY_ACCESS_GRANT (reason, expires_at ngắn)
    EHR->>EHR: Cấp quyền xem tạm thời, vượt qua giới hạn treatment relationship
    EHR-->>ClinicStaff: Trả về hồ sơ bệnh nhân X

    EHR->>EHR: Ghi ACCESS_LOG gắn với emergency_access_grant, đánh dấu rõ đây là truy cập khẩn cấp

    Note over EHR: Quyền tự động hết hạn sau expires_at
    EHR->>Reviewer: Gửi EMERGENCY_ACCESS_GRANT để review sau sự cố
    Reviewer->>EHR: Xác nhận hợp lệ hoặc gắn cờ điều tra thêm
```
