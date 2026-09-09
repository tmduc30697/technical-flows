# Base sequence — Xem lương (chỉ cách ly tenant, chưa cách ly nội bộ)

Đây là **base**, flow "Xem lương nhân viên" ở trạng thái hiện tại — chỉ kiểm tra `tenant_id` khớp (đúng công ty), sau đó cho phép xem lương bất kỳ đồng nghiệp nào trong cùng công ty mà không xét vai trò của người xem. Flow này liên quan mật thiết tới enhance vì yêu cầu 1 của đề bài chính là thêm lớp cách ly nội bộ theo vai trò còn thiếu ở đây.

```mermaid
sequenceDiagram
    actor Emp as Nhân viên thường (công ty A, phòng Kỹ thuật)
    participant App as Payroll Service
    participant DB as SALARY_RECORD store

    Emp->>App: Xem lương của đồng nghiệp B (cùng công ty A, phòng khác)
    App->>App: Kiểm tra tenant_id của Emp = tenant_id của nhân viên B
    App->>DB: SELECT * FROM salary_record WHERE employee_id = B
    DB-->>App: Bản ghi lương của B
    App-->>Emp: Trả về lương của B
    Note over App: Chỉ cách ly đúng tenant, không kiểm tra vai trò của Emp so với B — nhân viên thường xem được lương bất kỳ ai cùng công ty
```
