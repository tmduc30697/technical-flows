# Sequence Diagram — Enhance: Process Order Step

Đây là **enhance**, flow đã thay đổi so với base ([2-base-process-order-step-sqd.md](2-base-process-order-step-sqd.md)): thứ tự bắt buộc đảo ngược — ghi `WAL_ENTRY` trước, cập nhật in-memory cache sau, thay vì chỉ cập nhật cache như base. Sơ đồ cũng thể hiện lựa chọn giữa ghi WAL đồng bộ (an toàn tuyệt đối) và batch định kỳ (nhanh hơn, RPO khác 0).

```mermaid
sequenceDiagram
    actor OrderService as Order Processing Service
    participant WAL as WAL (disk)
    participant Cache as In-memory Cache

    OrderService->>OrderService: Nhận yêu cầu xử lý bước tiếp theo của đơn hàng

    alt write_mode = sync (an toàn tuyệt đối)
        OrderService->>WAL: Ghi WAL_ENTRY (order_id, step, status) + fsync ngay
        WAL-->>OrderService: Durable
        OrderService->>Cache: Update ORDER_PROCESSING_STATE, last_applied_lsn
        Cache-->>OrderService: Cập nhật xong
    else write_mode = batch (nhanh hơn, chấp nhận RPO nhỏ)
        OrderService->>Cache: Update ORDER_PROCESSING_STATE ngay để giữ tốc độ
        OrderService->>OrderService: Gom WAL_ENTRY vào buffer, flush định kỳ
        Note over OrderService,WAL: Nếu crash trước lần flush kế tiếp, các thay đổi gần nhất trong buffer có thể mất, đây là RPO chấp nhận được đã khai báo
    end
```
