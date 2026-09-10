# Sequence Diagram — Enhance: Single Logout

Đây là **enhance**, flow hoàn toàn mới: đăng xuất ở một app phải đăng xuất khỏi toàn bộ app con đã SSO vào, kể cả khi user chỉ đóng tab mà không bấm logout rõ ràng — không thể tồn tại ở base vì base không có khái niệm session dùng chung.

```mermaid
sequenceDiagram
    actor Employee as Nhân viên
    participant AppHR as HR Portal
    participant IdP as IdP nội bộ trung tâm
    participant AppWiki as Wiki nội bộ
    participant AppExpense as Công cụ báo cáo chi phí

    Employee->>AppHR: Bấm Đăng xuất
    AppHR->>IdP: Yêu cầu Single Logout cho SSO_SESSION hiện tại
    IdP->>IdP: Mark SSO_SESSION revoked_at=now

    par thông báo mọi app đã SSO vào
        IdP->>AppHR: Hủy APP_SESSION
        IdP->>AppWiki: Hủy APP_SESSION
        IdP->>AppExpense: Hủy APP_SESSION
    end

    Note over IdP,AppExpense: Kể cả khi user chỉ đóng tab Wiki mà không logout rõ ràng, APP_SESSION đó vẫn bị hủy theo SSO_SESSION trung tâm

    IdP-->>AppHR: Single Logout hoàn tất
    AppHR-->>Employee: Đã đăng xuất khỏi toàn bộ hệ thống
```
