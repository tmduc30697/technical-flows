# Sequence Diagram — Enhance: Clinic Contract Termination Revoke

Đây là **enhance**, flow hoàn toàn mới: khi phòng khám liên kết chấm dứt hợp đồng, toàn bộ quyền SSO của nhân viên phòng khám đó phải bị thu hồi ngay lập tức và đồng loạt, kể cả nhân viên đang có session hoạt động, tránh truy cập trái phép sau khi quan hệ đã chấm dứt.

```mermaid
sequenceDiagram
    actor Admin as Quản trị bệnh viện trung tâm
    participant Hospital as Bệnh viện trung tâm (IdP)
    participant EHR as Hệ thống EHR trung tâm

    Admin->>Hospital: Chấm dứt hợp đồng với Clinic Y
    Hospital->>Hospital: Update CLINIC_CONTRACT status=terminated, terminated_at=now

    Hospital->>Hospital: Tìm toàn bộ SSO_SESSION đang hoạt động của CLINIC_STAFF thuộc Clinic Y
    Hospital->>Hospital: Mark toàn bộ SSO_SESSION revoked_at=now, kể cả session đang hoạt động

    Hospital->>EHR: Thông báo thu hồi hàng loạt cho Clinic Y
    EHR->>EHR: Từ chối mọi request tiếp theo từ nhân viên Clinic Y

    Hospital-->>Admin: Xác nhận đã thu hồi toàn bộ quyền truy cập của Clinic Y
```
