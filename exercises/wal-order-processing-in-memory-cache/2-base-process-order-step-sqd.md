# Sequence Diagram — Base: Process Order Step

Đây là **base**, flow cập nhật trạng thái đơn hàng chỉ dựa vào in-memory cache, không có bất kỳ lớp ghi bền vững nào — tiền đề để thấy rõ rủi ro mất trạng thái xử lý khi service crash/restart.

```mermaid
sequenceDiagram
    actor OrderService as Order Processing Service
    participant Cache as In-memory Cache

    OrderService->>OrderService: Nhận yêu cầu xử lý bước tiếp theo của đơn hàng
    OrderService->>Cache: Update ORDER_PROCESSING_STATE (current_step, status)
    Cache-->>OrderService: Cập nhật xong, tốc độ rất nhanh
    OrderService-->>OrderService: Tiếp tục xử lý bước sau

    Note over OrderService,Cache: Nếu service crash/restart ngay sau bước này, toàn bộ trạng thái trong cache mất sạch, không có bản ghi nào để phục hồi
```
