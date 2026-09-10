# Sequence Diagram — Enhance: Embedded Partner SSO

Đây là **enhance**, flow hoàn toàn mới cho đối tác nhúng qua iframe/webview: cấp một token phạm vi giới hạn (`PARTNER_EMBED_GRANT`), không cho đối tác thấy hoặc lưu thông tin đăng nhập gốc, và Single Logout phải kết thúc luôn phiên nhúng này khi user đăng xuất ở nơi khác.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant Partner as App đối tác (iframe/webview)
    participant Auth as Auth Service (Fintech IdP)

    User->>Partner: Mở trải nghiệm nhúng của đối tác
    Partner->>Auth: Redirect SSO request (partner_id, scope yêu cầu)
    Auth->>Auth: Kiểm tra user đã có SESSION hợp lệ (web/mobile)

    alt đã đăng nhập ở kênh khác
        Auth-->>User: Xác nhận cấp quyền hạn chế cho đối tác (không lộ credential gốc)
        User->>Auth: Đồng ý cấp scope giới hạn
    else chưa đăng nhập
        Auth-->>User: Yêu cầu đăng nhập trước khi cấp quyền cho đối tác
        User->>Auth: Đăng nhập
    end

    Auth->>Auth: Tạo PARTNER_EMBED_GRANT (scope giới hạn, gắn với session gốc)
    Auth-->>Partner: Trả token phạm vi giới hạn, không phải session gốc
    Partner-->>User: Trải nghiệm nhúng hoạt động với quyền giới hạn

    Note over Auth,Partner: Đối tác không bao giờ thấy hoặc lưu được thông tin đăng nhập gốc của user
```
