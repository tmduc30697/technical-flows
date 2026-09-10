# Sequence Diagram — Enhance: Employee Offboarding Revoke

Đây là **enhance**, flow hoàn toàn mới: khi nhân viên nghỉ việc, revoke session ở IdP trung tâm phải làm mất quyền truy cập ở tất cả app con gần như ngay lập tức, không chờ từng app tự hết session như ở base.

```mermaid
sequenceDiagram
    participant HR as Hệ thống HR
    participant IdP as IdP nội bộ trung tâm
    participant AppHR as HR Portal
    participant AppWiki as Wiki nội bộ

    HR->>IdP: Đồng bộ nhân viên nghỉ việc (hr_status=terminated)
    IdP->>IdP: Tìm mọi SSO_SESSION đang hoạt động của employee
    IdP->>IdP: Mark toàn bộ SSO_SESSION revoked_at=now

    par vô hiệu hóa ngay tại các app
        IdP->>AppHR: Hủy APP_SESSION liên quan
        IdP->>AppWiki: Hủy APP_SESSION liên quan
    end

    IdP->>IdP: Ghi AUDIT_LOG (action=offboarding_revoke)
    IdP-->>HR: Xác nhận đã thu hồi toàn bộ quyền truy cập
```
