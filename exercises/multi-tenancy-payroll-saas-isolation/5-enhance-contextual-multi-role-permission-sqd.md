# Enhance sequence — Một người có vai trò khác nhau theo từng ngữ cảnh phòng ban

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base gán 1 vai trò cố định cho mỗi người nên không thể diễn tả trường hợp này. Đáp ứng yêu cầu 2 của đề bài: quyền xem dữ liệu phải tính đúng theo ngữ cảnh cụ thể (phòng ban nào), không cấp nhầm quyền cao nhất của người đó cho mọi ngữ cảnh.

```mermaid
sequenceDiagram
    actor Manager as Người dùng M (quản lý phòng A, đồng thời là thành viên thường của dự án liên phòng ban B)
    participant App as Payroll Service
    participant Role as DEPARTMENT_ROLE_ASSIGNMENT
    participant Log as SALARY_ACCESS_LOG

    Manager->>App: Xem lương của nhân viên X (thuộc phòng A, M là quản lý phòng A)
    App->>Role: Tìm assignment của M tại department_id = A
    Role-->>App: DEPARTMENT_ROLE_ASSIGNMENT(role=manager, assignment_type=home_department)
    App->>Log: Ghi SALARY_ACCESS_LOG(decision=allowed, reason=manager_of_department)
    App-->>Manager: Cho phép xem lương của X

    Manager->>App: Xem lương của nhân viên Y (thành viên dự án liên phòng ban B, M cũng tham gia dự án B nhưng chỉ với vai trò thành viên thường)
    App->>Role: Tìm assignment của M tại ngữ cảnh dự án B
    Role-->>App: DEPARTMENT_ROLE_ASSIGNMENT(role=employee, assignment_type=cross_department_project)
    Note over App: M có vai trò manager ở phòng A, nhưng ở ngữ cảnh dự án B chỉ là employee — không được cộng dồn hay mượn quyền cao nhất từ ngữ cảnh khác
    App->>Log: Ghi SALARY_ACCESS_LOG(decision=denied, reason=denied_no_context_role)
    App-->>Manager: Từ chối xem lương của Y, vì ở đúng ngữ cảnh này M chỉ là nhân viên thường
```
