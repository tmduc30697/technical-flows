# Base sequence — Login

Đây là **base**, flow "Đăng nhập bằng username/password" — tiền đề trực tiếp cho enhance: chính vì hệ thống xác thực bằng mật khẩu tĩnh và cấm self-service reset, nên khi nhân viên quên mật khẩu, họ hoàn toàn phụ thuộc vào helpdesk (vấn đề mà đề bài yêu cầu xây quy trình xử lý).

```mermaid
sequenceDiagram
    actor Employee
    participant App as Document System
    participant DB as EMPLOYEE store

    Employee->>App: Nhập username + password
    App->>DB: Tra password_hash theo username
    DB-->>App: Trả về password_hash
    App->>App: So khớp hash
    App->>App: Tạo SESSION mới
    App-->>Employee: Đăng nhập thành công, vào hệ thống tài liệu
```
