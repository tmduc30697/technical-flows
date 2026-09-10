# Sequence Diagram — Base: App Login

Đây là **base**, flow nhân viên đăng nhập riêng lẻ vào từng app nội bộ với tài khoản/session cục bộ của app đó — tiền đề bắt buộc để enhance thay thế bằng luồng SSO tập trung, vì "đăng nhập một lần, truy cập mọi app" chỉ có nghĩa khi trước đó mỗi app đang login độc lập.

```mermaid
sequenceDiagram
    actor Employee as Nhân viên
    participant AppHR as HR Portal
    participant AppWiki as Wiki nội bộ

    Employee->>AppHR: Đăng nhập (email/password riêng của HR Portal)
    AppHR->>AppHR: Kiểm tra APP_ACCOUNT, tạo APP_SESSION
    AppHR-->>Employee: Truy cập HR Portal

    Employee->>AppWiki: Truy cập Wiki nội bộ
    AppWiki->>AppWiki: Chưa có session, yêu cầu đăng nhập riêng
    Employee->>AppWiki: Đăng nhập lại (email/password riêng của Wiki)
    AppWiki-->>Employee: Truy cập Wiki, session tách biệt hoàn toàn với HR Portal
```
