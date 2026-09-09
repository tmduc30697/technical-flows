# Sequence - Enhance - Flow "concurrent-order-conflict"

Đây là **enhance**, flow hoàn toàn mới so với base, mô tả chi tiết kịch bản race mà yêu cầu 2 nêu ra: cache nội bộ báo còn 1, 2 khách cùng đặt mua nên cả 2 đều trừ cache thành công (vì cache đã lệch so với nhà cung cấp chỉ còn 1 thật). Quy tắc xử lý: request xác nhận với nhà cung cấp nào đến trước được ưu tiên giữ, request sau khi biết nhà cung cấp báo hết phải tự động hủy, hoàn tiền và giải thích rõ nguyên nhân cho khách thứ 2.

```mermaid
sequenceDiagram
    actor Customer1 as Khách A
    actor Customer2 as Khách B
    participant App as Sàn dropshipping
    participant Cache as Inventory Cache
    participant SupplierAPI as API Nhà cung cấp

    Note over Cache: cached_quantity = 1, nhưng nhà cung cấp thực tế cũng chỉ còn 1

    par Khách A đặt mua
        Customer1->>App: đặt mua sản phẩm X
        App->>Cache: trừ tạm reserved_count, thành công (cache cho phép)
        App->>App: tạo order A, status = pending_confirmation
    and Khách B đặt mua gần như cùng lúc
        Customer2->>App: đặt mua sản phẩm X
        App->>Cache: trừ tạm reserved_count, thành công (cache cho phép vì lệch thực tế)
        App->>App: tạo order B, status = pending_confirmation
    end

    Note over App,SupplierAPI: Cả 2 order cùng vào bước xác nhận thật, order A tới trước

    App->>SupplierAPI: xác nhận đặt hàng thật cho order A
    SupplierAPI-->>App: còn hàng thật, xác nhận thành công
    App->>App: order A.status = confirmed
    App-->>Customer1: đặt hàng thành công

    App->>SupplierAPI: xác nhận đặt hàng thật cho order B
    SupplierAPI-->>App: hết hàng thật (đã bị order A giữ)
    App->>Cache: rollback reserved_count cho order B
    App->>App: order B.status = cancelled_out_of_stock
    App-->>Customer2: hủy đơn, hoàn tiền, giải thích rõ nguyên nhân\nlà lệch dữ liệu tồn kho từ nhà cung cấp, không phải lỗi thao tác của khách
```
