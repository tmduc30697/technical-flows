# Sequence - Base - Flow "place-order"

Đây là **base**: khách đặt mua chỉ dựa vào tồn kho cache nội bộ, không có bước xác nhận thật với nhà cung cấp trước khi chốt đơn. Flow này được chọn vì nó là hiện trạng mà enhance yêu cầu sửa: chưa trừ cache nguyên tử, chưa gọi API xác nhận, chưa có rollback khi nhà cung cấp thực tế đã hết hàng (yêu cầu 1).

```mermaid
sequenceDiagram
    actor Customer
    participant App as Sàn dropshipping
    participant Cache as Inventory Cache

    Customer->>App: đặt mua sản phẩm X
    App->>Cache: đọc cached_quantity
    Cache-->>App: cached_quantity = 1

    App-->>App: kiểm tra cached_quantity > 0, hợp lệ
    App->>Cache: ghi cached_quantity = 0
    App->>App: tạo order, status = confirmed ngay lập tức
    App-->>Customer: đặt hàng thành công

    Note over App,Cache: Không có bước gọi nhà cung cấp để xác nhận tồn kho thật,\nnếu cache đã lệch thì khách vẫn được báo thành công dù thực tế hết hàng
```
