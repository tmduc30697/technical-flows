# Enhance ERD — sau khi có quy trình break-glass do helpdesk xử lý

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 thay đổi chính, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `EMPLOYEE` thêm `must_change_password` và `mfa_enrolled` — buộc đổi mật khẩu tạm ở lần đăng nhập kế tiếp, và bắt buộc MFA cho agent trước khi thực hiện reset.
- `RESET_TICKET` (mới) — mọi yêu cầu reset đều phải đi qua ticket gắn xác minh danh tính (qua quản lý trực tiếp), thay vì xử lý miệng như base; có `requires_second_approver`/`secondary_agent_id` để áp dụng four-eyes cho tài khoản quyền cao.
- `AUDIT_LOG` (mới) — ghi nhận agent nào đã reset cho tài khoản nào.
- `ANOMALY_ALERT` (mới) — phát hiện agent reset số lượng tài khoản bất thường trong thời gian ngắn.

```mermaid
erDiagram
    EMPLOYEE ||--o{ SESSION : creates
    EMPLOYEE ||--o{ EMPLOYEE : "quản lý trực tiếp (manager_id)"
    EMPLOYEE ||--o{ RESET_TICKET : "là đối tượng được reset"
    EMPLOYEE ||--o{ RESET_TICKET : "agent chính thực hiện"
    EMPLOYEE ||--o{ RESET_TICKET : "agent thứ hai duyệt (four-eyes)"
    RESET_TICKET ||--o{ AUDIT_LOG : generates
    EMPLOYEE ||--o{ ANOMALY_ALERT : "bị gắn cảnh báo (agent)"

    EMPLOYEE {
        string id PK
        string username
        string password_hash
        string role "employee | helpdesk | admin"
        boolean is_privileged
        string manager_id FK
        boolean must_change_password
        boolean mfa_enrolled
    }
    SESSION {
        string id PK
        string employee_id FK
        datetime created_at
        datetime expires_at
    }
    RESET_TICKET {
        string id PK
        string target_employee_id FK
        string manager_verification_status
        string primary_agent_id FK
        string primary_agent_mfa_status
        boolean requires_second_approver
        string secondary_agent_id FK
        string secondary_agent_mfa_status
        string status
        datetime created_at
        datetime completed_at
    }
    AUDIT_LOG {
        string id PK
        string ticket_id FK
        string agent_id FK
        string action
        datetime created_at
    }
    ANOMALY_ALERT {
        string id PK
        string agent_id FK
        datetime window_start
        datetime window_end
        int reset_count
        int threshold
        string status
    }
```
