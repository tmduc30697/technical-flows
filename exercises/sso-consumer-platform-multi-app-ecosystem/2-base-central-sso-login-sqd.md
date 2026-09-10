# Sequence Diagram — Base: Central SSO Login

Đây là **base**, flow đăng nhập một lần tại IdP trung tâm rồi mọi app con tin tưởng thẳng token trả về, không tự truy vấn quyền riêng — đây chính là điểm sẽ thay đổi ở enhance vì token không thể mang theo toàn bộ quyền của mọi app.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant IdP as Central IdP
    participant AppA as App con (vd Storage App)

    User->>IdP: Đăng nhập (email/password)
    IdP->>IdP: Xác thực, tạo central_token
    IdP-->>User: central_token
    User->>AppA: Truy cập kèm central_token
    AppA->>AppA: Tin tưởng token, cho phép truy cập đầy đủ
    AppA-->>User: Truy cập thành công
```
