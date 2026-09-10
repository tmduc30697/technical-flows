# Sequence Diagram — Enhance: Per-App Revoke

Đây là **enhance**, flow hoàn toàn mới cho phép user thu hồi quyền truy cập của một app cụ thể mà không ảnh hưởng các app khác trong hệ sinh thái, thông qua việc revoke `APP_GRANT` theo phạm vi thay vì đăng xuất toàn bộ hệ sinh thái.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant IdP as Central IdP
    participant AppA as App con muốn ngắt kết nối
    participant AppB as App con khác

    User->>IdP: Yêu cầu revoke quyền truy cập của App A
    IdP->>IdP: Tìm APP_GRANT của user cho App A
    IdP->>IdP: Đánh dấu APP_GRANT.revoked_at=now cho App A
    IdP->>AppA: Thông báo grant bị revoke
    AppA->>AppA: Vô hiệu hóa mọi APP_SESSION liên quan tới grant này

    Note over AppB: App B không bị ảnh hưởng, session vẫn hoạt động bình thường
    IdP-->>User: Đã ngắt kết nối App A, các app khác vẫn hoạt động
```
