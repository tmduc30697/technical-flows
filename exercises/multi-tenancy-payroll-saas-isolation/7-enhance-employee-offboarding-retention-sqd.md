# Enhance sequence — Offboarding nhân viên, giữ đúng ranh giới tenant

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base không có quy trình offboarding, chỉ có `status = active` cố định. Đáp ứng yêu cầu 4 của đề bài: khi công ty khách hàng offboard nhân viên, dữ liệu lương/hồ sơ vẫn phải giữ cách ly đúng theo tenant đó cho mục đích lưu trữ pháp luật lao động, không bị gộp lẫn hay mất ranh giới tenant theo thời gian.

```mermaid
sequenceDiagram
    actor HR as HR công ty A
    participant App as Payroll Service
    participant Emp as EMPLOYEE store
    participant Event as EMPLOYEE_OFFBOARDING_EVENT
    participant DB as SALARY_RECORD store
    participant Log as SALARY_ACCESS_LOG

    HR->>App: Offboard nhân viên X (nghỉ việc tại công ty A)
    App->>Emp: Cập nhật EMPLOYEE(X).status=offboarded, offboarded_at=now
    App->>Event: Tạo EMPLOYEE_OFFBOARDING_EVENT(employee_id=X, tenant_id=A, reason, retention_note)
    Note over DB: SALARY_RECORD của X vẫn giữ nguyên tenant_id=A, không xóa, không di chuyển sang kho lưu trữ dùng chung nhiều tenant

    HR->>App: Nhiều năm sau, tra cứu lại hồ sơ lương của X phục vụ thanh tra lao động
    App->>App: Kiểm tra tenant_id(HR) = A = tenant_id(X) như bình thường, dù X đã offboard
    App->>DB: SELECT * FROM salary_record WHERE employee_id = X
    DB-->>App: Vẫn trả đúng dữ liệu, vẫn gắn tenant_id=A
    App->>Log: Ghi SALARY_ACCESS_LOG(decision=allowed, reason=hr_company_wide, target_tenant=A)
    App-->>HR: Trả về hồ sơ lương lịch sử của X, ranh giới tenant A không đổi theo thời gian

    Note over App: Nhân viên đã offboard không tự đăng nhập xem lại được, chỉ HR/admin còn hoạt động của đúng tenant A mới tra cứu được
```
