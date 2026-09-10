# Sequence Diagram — Enhance: Generate Audit Trail

Đây là **enhance**, flow mới xử lý yêu cầu log đủ thông tin để replay ra audit trail đầy đủ, phục vụ đối soát với ngân hàng đối tác/cơ quan quản lý sau này.

```mermaid
sequenceDiagram
    actor Auditor as Ops/Regulator
    participant AuditSvc as Audit Service
    participant WAL_A as WAL Disk/Node A
    participant DB as Database

    Auditor->>AuditSvc: Yêu cầu audit trail cho khoảng thời gian X
    AuditSvc->>WAL_A: Đọc toàn bộ WAL_ENTRY trong khoảng thời gian đó theo thứ tự lsn
    WAL_A-->>AuditSvc: Danh sách entry (transaction_id, account_id, entry_type, amount, checksum, written_at)

    AuditSvc->>AuditSvc: Verify checksum từng entry, loại bỏ entry không hợp lệ khỏi báo cáo
    AuditSvc->>DB: Đối chiếu số dư hiện tại với tổng hợp từ WAL_ENTRY
    DB-->>AuditSvc: Khớp hoặc phát hiện sai lệch

    AuditSvc-->>Auditor: Audit trail đầy đủ, có thể dùng để đối soát với đối tác/cơ quan quản lý
```
