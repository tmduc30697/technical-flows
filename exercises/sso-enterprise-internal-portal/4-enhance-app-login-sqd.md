# Sequence Diagram — Enhance: App Login

Đây là **enhance**, flow app login thay đổi hoàn toàn so với base: nhân viên chỉ đăng nhập một lần tại IdP trung tâm để tạo `SSO_SESSION`, sau đó mọi app khác trên các subdomain khác nhau tự nhận `APP_SESSION` từ session trung tâm mà không cần đăng nhập lại, kèm cơ chế chống session fixation khi chuyển giữa các app.

```mermaid
sequenceDiagram
    actor Employee as Nhân viên
    participant IdP as IdP nội bộ trung tâm
    participant AppHR as HR Portal
    participant AppWiki as Wiki nội bộ

    Employee->>AppHR: Truy cập HR Portal
    AppHR->>IdP: Chưa có SSO_SESSION, redirect đăng nhập
    Employee->>IdP: Đăng nhập (email/password)
    IdP->>IdP: Tạo SSO_SESSION mới (session_token mới, chống session fixation)
    IdP-->>AppHR: Redirect về kèm session_token
    AppHR->>AppHR: Tạo APP_SESSION từ session_token, ghi AUDIT_LOG
    AppHR-->>Employee: Truy cập HR Portal

    Employee->>AppWiki: Truy cập Wiki nội bộ (subdomain khác)
    AppWiki->>IdP: Kiểm tra SSO_SESSION hiện có
    IdP-->>AppWiki: Session hợp lệ, employee_id
    AppWiki->>AppWiki: Tạo APP_SESSION mới, không cần đăng nhập lại, ghi AUDIT_LOG
    AppWiki-->>Employee: Truy cập Wiki ngay lập tức
```
