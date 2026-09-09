# Enhance sequence — Privileged account reset (four-eyes)

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đáp ứng yêu cầu thứ 4 của đề bài: tài khoản quyền cao (`is_privileged = true`, vd admin hệ thống tài liệu) phải có **2 agent** xác nhận độc lập trước khi reset, không cho 1 agent đơn lẻ tự thực hiện.

```mermaid
sequenceDiagram
    actor Employee as Privileged Employee
    actor Agent1 as Helpdesk Agent (chính)
    actor Manager
    actor Agent2 as Helpdesk Agent/Supervisor (thứ hai)
    participant DB as RESET_TICKET store
    participant Audit as AUDIT_LOG

    Employee->>Agent1: Báo quên mật khẩu qua ticket helpdesk
    Agent1->>DB: Tạo RESET_TICKET (target=employee, status=pending_verification)
    Agent1->>Manager: Yêu cầu xác nhận danh tính
    Manager-->>DB: manager_verification_status=verified
    Agent1->>Agent1: Xác thực MFA của agent chính
    Agent1->>DB: Kiểm tra employee.is_privileged = true → requires_second_approver=true
    DB-->>Agent1: Ticket chuyển status=pending_second_approval, chưa cho reset
    DB->>Agent2: Thông báo cần duyệt lần 2 cho ticket này
    Agent2->>Agent2: Độc lập xác minh lại danh tính + bối cảnh ticket
    Agent2->>Agent2: Xác thực MFA của agent thứ hai
    Agent2->>DB: Xác nhận duyệt (secondary_agent_id, secondary_agent_mfa_status=verified)
    DB->>DB: Cả 2 điều kiện đủ → cho phép thực hiện reset
    DB->>DB: Đặt password_hash tạm, must_change_password = true
    DB->>Audit: Ghi log cả 2 agent (agent_id chính + agent_id thứ hai, action=reset)
    DB-->>Employee: Cấp mật khẩu tạm qua kênh bảo mật
```
