# Sequence Diagram — Base: Write Employee Data

Đây là **base**, flow ghi dữ liệu nhân sự (ví dụ sửa lương) trong mô hình shared-schema hiện tại — tiền đề cho việc migrate: chính flow ghi trực tiếp, không qua routing, này là thứ mà flow migration phải thay thế bằng cơ chế tra bảng routing trước khi ghi.

```mermaid
sequenceDiagram
    actor HRUser as HR User
    participant App as HR App Service
    participant DB as Shared Schema (public)

    HRUser->>App: Update employee salary (tenant_id, employee_id, new_amount)
    App->>DB: UPDATE SALARY_RECORD WHERE tenant_id=? AND employee_id=?
    DB-->>App: Row updated

    App-->>HRUser: Salary updated

    Note over App,DB: Schema is hardcoded in application config
    Note over App,DB: no notion of "which schema is source of truth for this tenant"
```
