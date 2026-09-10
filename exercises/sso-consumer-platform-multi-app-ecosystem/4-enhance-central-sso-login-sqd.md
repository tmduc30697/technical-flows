# Sequence Diagram — Enhance: Central SSO Login

Đây là **enhance**, flow login thay đổi so với base: sau khi IdP xác thực danh tính, mỗi app con tự truy vấn `APP_PERMISSION` riêng của mình thay vì tin token mang sẵn toàn quyền, đồng thời user phải chọn rõ ngữ cảnh cá nhân/tổ chức nếu tài khoản thuộc cả hai.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant IdP as Central IdP
    participant AppA as App con (vd Storage App)

    User->>IdP: Đăng nhập (email/password)
    IdP->>IdP: Xác thực danh tính
    alt tài khoản có cả ngữ cảnh cá nhân và tổ chức
        IdP-->>User: Yêu cầu chọn ngữ cảnh (cá nhân/workspace công ty)
        User->>IdP: Chọn ngữ cảnh
    end
    IdP->>IdP: Tạo central_token (chỉ mang danh tính + context_id, không mang quyền app)
    IdP-->>User: central_token

    User->>AppA: Truy cập kèm central_token
    AppA->>IdP: Xác minh token hợp lệ
    IdP-->>AppA: Token hợp lệ, context_id
    AppA->>AppA: Tự truy vấn APP_PERMISSION riêng của app này cho user + context
    AppA-->>User: Truy cập với đúng vai trò của app này, ví dụ admin ở Storage App
```
