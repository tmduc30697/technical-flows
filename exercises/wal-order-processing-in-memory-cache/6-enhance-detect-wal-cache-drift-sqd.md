# Sequence Diagram — Enhance: Detect WAL-Cache Drift

Đây là **enhance**, flow mới xử lý yêu cầu phát hiện WAL và in-memory cache bị lệch nhau (ví dụ do bug ghi cache thất bại âm thầm dù WAL đã ghi đúng), thông qua kiểm tra định kỳ/checksum trạng thái tổng hợp.

```mermaid
sequenceDiagram
    participant Scheduler as Consistency Check Scheduler
    participant OrderService as Order Processing Service
    participant WAL as WAL (disk)
    participant Cache as In-memory Cache
    actor Ops as Ops Team

    Scheduler->>OrderService: Trigger kiểm tra định kỳ
    OrderService->>WAL: Tính checksum trạng thái tổng hợp từ WAL_ENTRY mới nhất mỗi order
    OrderService->>Cache: Tính checksum trạng thái hiện tại trong cache

    OrderService->>OrderService: So sánh wal_derived_checksum và cache_checksum

    alt khớp nhau
        OrderService->>OrderService: Ghi CONSISTENCY_CHECK, matched = true
    else lệch nhau
        OrderService->>OrderService: Ghi CONSISTENCY_CHECK, matched = false
        OrderService->>Ops: Cảnh báo lệch dữ liệu, chỉ rõ order_id bị ảnh hưởng
        Ops->>OrderService: Điều tra, có thể force rebuild lại cache của order đó từ WAL
    end
```
