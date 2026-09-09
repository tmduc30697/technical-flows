# Enhance sequence — Login

Đây là **enhance**, cùng flow "Login" đã có ở base nhưng nay thay đổi theo yêu cầu thứ 2 của đề bài: sau khi xác thực bằng mật khẩu tạm do helpdesk cấp, hệ thống kiểm tra `must_change_password` và **buộc đổi mật khẩu ngay** trước khi cấp session đầy đủ — không cho phép dùng mật khẩu tạm lâu dài như base.

```mermaid
sequenceDiagram
    actor Employee
    participant App as Document System
    participant DB as EMPLOYEE store

    Employee->>App: Nhập username + mật khẩu tạm
    App->>DB: Tra password_hash + must_change_password
    DB-->>App: Khớp hash, must_change_password = true
    App-->>Employee: Chuyển hướng bắt buộc sang màn hình đổi mật khẩu
    Employee->>App: Nhập mật khẩu mới
    App->>DB: Cập nhật password_hash, must_change_password = false
    App->>App: Tạo SESSION đầy đủ
    App-->>Employee: Vào hệ thống tài liệu
```
