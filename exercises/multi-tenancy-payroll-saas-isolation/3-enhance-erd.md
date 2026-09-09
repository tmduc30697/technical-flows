# Enhance ERD — sau khi có cách ly 2 lớp (tenant + vai trò theo ngữ cảnh)

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm thay đổi/entity mới, ứng trực tiếp với 4 yêu cầu trong đề bài:

- `DEPARTMENT_ROLE_ASSIGNMENT` (mới, thay cho `EMPLOYEE.role` cố định) — vai trò được gắn theo từng ngữ cảnh phòng ban, cho phép 1 người có vai trò khác nhau ở các phòng ban khác nhau (vd quản lý phòng A, chỉ là thành viên dự án liên phòng ban B).
- `PAYROLL_SUMMARY_REPORT` (mới) — báo cáo tổng hợp có `sample_size`/`min_sample_size`/`suppressed` để chặn suy luận ngược khi phòng ban chỉ có 1 người.
- `EMPLOYEE` (sửa) + `EMPLOYEE_OFFBOARDING_EVENT` (mới) — mở rộng `status` thêm `offboarded`, ghi nhận thời điểm và lý do nghỉ việc, dữ liệu vẫn giữ đúng ranh giới `tenant_id` cũ.
- `SALARY_ACCESS_LOG` (mới) — audit chi tiết mọi lần truy cập lương, ai xem, khi nào, tenant nào xem tenant nào.

```mermaid
erDiagram
    TENANT ||--o{ EMPLOYEE : employs
    TENANT ||--o{ DEPARTMENT : has
    DEPARTMENT ||--o{ EMPLOYEE : "home department"
    EMPLOYEE ||--o{ SALARY_RECORD : has
    EMPLOYEE ||--o{ DEPARTMENT_ROLE_ASSIGNMENT : "has (theo từng ngữ cảnh)"
    DEPARTMENT ||--o{ DEPARTMENT_ROLE_ASSIGNMENT : "scoped to"
    EMPLOYEE ||--o{ SALARY_ACCESS_LOG : "viewed by (as viewer)"
    SALARY_RECORD ||--o{ SALARY_ACCESS_LOG : "accessed via"
    EMPLOYEE ||--o| EMPLOYEE_OFFBOARDING_EVENT : "may have"
    DEPARTMENT ||--o{ PAYROLL_SUMMARY_REPORT : "aggregated for"

    TENANT {
        string id PK
        string name
    }
    DEPARTMENT {
        string id PK
        string tenant_id FK
        string name
    }
    EMPLOYEE {
        string id PK
        string tenant_id FK
        string department_id FK "home department, chỉ để tham chiếu tổ chức"
        string name
        string email
        string status "active | offboarded"
        datetime offboarded_at "nullable"
    }
    DEPARTMENT_ROLE_ASSIGNMENT {
        string id PK
        string employee_id FK
        string department_id FK
        string role "employee | manager | hr | admin"
        string assignment_type "home_department | cross_department_project"
    }
    SALARY_RECORD {
        string id PK
        string tenant_id FK
        string employee_id FK
        decimal amount
        date effective_date
    }
    SALARY_ACCESS_LOG {
        string id PK
        string viewer_employee_id FK
        string viewer_tenant_id
        string target_employee_id FK
        string target_tenant_id
        string decision "allowed | denied"
        string reason "vd self_view | manager_of_department | hr_company_wide | denied_no_context_role"
        datetime accessed_at
    }
    EMPLOYEE_OFFBOARDING_EVENT {
        string id PK
        string employee_id FK
        string tenant_id FK
        string reason
        datetime offboarded_at
        string retention_note "giữ đúng ranh giới tenant cho mục đích lưu trữ pháp lý"
    }
    PAYROLL_SUMMARY_REPORT {
        string id PK
        string tenant_id FK
        string department_id FK
        decimal avg_salary
        int sample_size
        int min_sample_size "vd 3"
        boolean suppressed "true nếu sample_size < min_sample_size"
        datetime computed_at
    }
```
