# Enhance sequence — Password reset (tài khoản thường)

Đây là **enhance**, cùng flow "Password reset" đã có ở base nhưng nay thay đổi hoàn toàn theo yêu cầu 1-3 của đề bài: không còn xử lý miệng, mà bắt buộc qua `RESET_TICKET` gắn xác minh qua quản lý trực tiếp, agent phải xác thực MFA trước khi thực hiện, và mọi thao tác được ghi vào `AUDIT_LOG`. Áp dụng cho tài khoản **không** phải quyền cao (`is_privileged = false`) — tài khoản quyền cao xem flow "Privileged account reset" riêng.

```mermaid
sequenceDiagram
    actor Employee
    actor Agent as Helpdesk Agent
    actor Manager
    participant DB as RESET_TICKET / EMPLOYEE store
    participant Audit as AUDIT_LOG

    Employee->>Agent: Tạo ticket báo quên mật khẩu (kênh helpdesk chính thức, không có endpoint self-service)
    Agent->>DB: Tạo RESET_TICKET (target=employee, status=pending_verification)
    Agent->>Manager: Yêu cầu xác nhận danh tính nhân viên
    Manager-->>DB: Xác nhận danh tính (manager_verification_status=verified)
    Agent->>Agent: Xác thực MFA của chính agent
    Agent->>DB: Kiểm tra employee.is_privileged = false → không cần second approver
    DB-->>Agent: Cho phép thực hiện reset (đủ điều kiện: verified + MFA + không privileged)
    Agent->>DB: Đặt password_hash tạm, must_change_password = true
    DB->>Audit: Ghi log (ticket_id, agent_id, action=reset, timestamp)
    DB-->>Agent: RESET_TICKET status=completed
    Agent-->>Employee: Cấp mật khẩu tạm qua kênh bảo mật (buộc đổi ở lần đăng nhập kế tiếp)
```
