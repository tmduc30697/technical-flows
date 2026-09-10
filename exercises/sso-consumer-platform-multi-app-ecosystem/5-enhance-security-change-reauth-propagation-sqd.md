# Sequence Diagram — Enhance: Security Change Reauth Propagation

Đây là **enhance**, flow hoàn toàn mới: khi user đổi thông tin nhạy cảm ở tài khoản trung tâm, mọi app con đang có session hoạt động phải nhận tín hiệu để yêu cầu xác thực lại, tránh trường hợp kẻ tấn công chiếm tài khoản trung tâm vẫn dùng được session cũ ở app khác vô thời hạn.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant IdP as Central IdP
    participant AppA as App con A
    participant AppB as App con B

    User->>IdP: Đổi email/password hoặc bật thêm MFA
    IdP->>IdP: Cập nhật thông tin, tạo SECURITY_EVENT (event_type)
    IdP->>AppA: Phát tín hiệu security event (webhook/push)
    IdP->>AppB: Phát tín hiệu security event (webhook/push)

    AppA->>AppA: Đánh dấu APP_SESSION hiện tại reauth_required=true
    AppB->>AppB: Đánh dấu APP_SESSION hiện tại reauth_required=true

    Note over AppA,AppB: Session cũ không bị hủy ngay, nhưng thao tác nhạy cảm tiếp theo sẽ yêu cầu xác thực lại

    User->>AppA: Thao tác tiếp theo trên App A
    AppA-->>User: Yêu cầu xác thực lại qua IdP trước khi tiếp tục
```
