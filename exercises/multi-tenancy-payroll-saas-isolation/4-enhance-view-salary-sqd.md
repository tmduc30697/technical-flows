# Enhance sequence — Xem lương (cách ly 2 lớp: tenant + vai trò, có audit log)

Đây là **enhance**, cùng flow "Xem lương nhân viên" đã có ở base nhưng nay thay đổi theo yêu cầu 1 và 5 của đề bài: sau khi qua lớp ngoài (tenant_id khớp), phải qua thêm lớp trong kiểm tra vai trò của người xem đối với người bị xem (tự xem, quản lý đúng phòng ban, HR/admin công ty), và mọi lần truy cập đều được ghi log chi tiết.

```mermaid
sequenceDiagram
    actor Emp as Nhân viên thường (công ty A, phòng Kỹ thuật)
    participant App as Payroll Service
    participant Role as DEPARTMENT_ROLE_ASSIGNMENT
    participant DB as SALARY_RECORD store
    participant Log as SALARY_ACCESS_LOG

    Emp->>App: Xem lương của đồng nghiệp B (cùng công ty A, phòng Kinh doanh)
    App->>App: Lớp ngoài, kiểm tra tenant_id(Emp) = tenant_id(B)
    App->>Role: Lớp trong, tìm DEPARTMENT_ROLE_ASSIGNMENT của Emp tại department_id(B)
    Role-->>App: Không tìm thấy assignment nào của Emp tại phòng Kinh doanh
    App->>Log: Ghi SALARY_ACCESS_LOG(decision=denied, reason=denied_no_context_role)
    App-->>Emp: Từ chối, không có vai trò quản lý/hr/admin tại phòng ban của B

    Emp->>App: Xem lương của chính mình
    App->>App: Lớp ngoài hợp lệ, lớp trong: self_view luôn được phép
    App->>DB: SELECT * FROM salary_record WHERE employee_id = Emp
    DB-->>App: Bản ghi lương của Emp
    App->>Log: Ghi SALARY_ACCESS_LOG(decision=allowed, reason=self_view)
    App-->>Emp: Trả về lương của chính mình
```
