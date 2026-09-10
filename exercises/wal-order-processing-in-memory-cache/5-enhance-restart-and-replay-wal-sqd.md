# Sequence Diagram — Enhance: Restart and Replay WAL

Đây là **enhance**, flow mới hoàn toàn: khi service restart, phải replay WAL rebuild lại toàn bộ in-memory cache về đúng trạng thái trước crash, xử lý đúng theo entry cuối cùng của từng đơn hàng, trước khi bắt đầu nhận request xử lý đơn mới.

```mermaid
sequenceDiagram
    participant OrderService as Order Processing Service (restarted)
    participant WAL as WAL (disk)
    participant Cache as In-memory Cache

    OrderService->>OrderService: Restart, cache đang rỗng
    OrderService->>OrderService: Chưa nhận request xử lý đơn mới nào (chặn tạm)

    OrderService->>WAL: Đọc toàn bộ WAL_ENTRY theo thứ tự lsn

    loop nhóm theo order_id
        OrderService->>OrderService: Chỉ áp dụng entry cuối cùng ghi nhận cho order đó
        OrderService->>Cache: Rebuild ORDER_PROCESSING_STATE đúng theo entry cuối
    end

    Cache-->>OrderService: Toàn bộ cache đã rebuild xong, khớp trạng thái trước crash
    OrderService->>OrderService: Mở nhận request xử lý đơn mới
    Note over OrderService,Cache: Nếu nhận request khi cache còn rỗng/chưa replay xong sẽ đọc sai trạng thái đơn hàng
```
