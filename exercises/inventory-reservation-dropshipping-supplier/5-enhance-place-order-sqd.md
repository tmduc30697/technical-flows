# Sequence - Enhance - Flow "place-order"

Đây là **enhance**, cùng flow `place-order` như ở base nhưng nay gồm 3 bước bắt buộc theo yêu cầu 1: (1) trừ tạm cache nguyên tử, (2) gọi API xác nhận đặt hàng thật với nhà cung cấp, (3) rollback cache và hoàn tiền nếu nhà cung cấp báo hết hàng thật. Bước gọi nhà cung cấp cũng áp dụng chính sách retry/timeout theo yêu cầu 4: tối đa 3 lần thử trong 90 giây, quá đó coi là thất bại và tự động hoàn tiền, tránh đơn ở trạng thái "đang xử lý" vô thời hạn.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Sàn dropshipping
    participant Cache as Inventory Cache
    participant SupplierAPI as API Nhà cung cấp

    Customer->>App: đặt mua sản phẩm X

    App->>Cache: UPDATE product SET reserved_count = reserved_count + 1\nWHERE id = X AND cached_quantity - reserved_count >= 1
    Cache-->>App: 1 dòng bị ảnh hưởng, trừ tạm thành công
    App->>App: tạo order, status = pending_confirmation

    loop Tối đa 3 lần thử, trong vòng 90 giây
        App->>SupplierAPI: xác nhận đặt hàng thật cho order
        alt Nhà cung cấp phản hồi kịp thời
            SupplierAPI-->>App: kết quả xác nhận (còn hàng hoặc hết hàng)
        else Timeout lần thử này
            App-->>App: ghi nhận attempt thất bại do timeout, thử lại nếu còn lượt
        end
    end

    alt Nhà cung cấp xác nhận còn hàng thật
        App->>App: order.status = confirmed
        App-->>Customer: đặt hàng thành công
    else Nhà cung cấp báo hết hàng thật
        App->>Cache: rollback, reserved_count -= 1
        App->>App: order.status = cancelled_out_of_stock
        App-->>Customer: thông báo hủy đơn, hoàn tiền
    else Hết số lần retry vẫn timeout
        App->>Cache: rollback, reserved_count -= 1
        App->>App: order.status = cancelled_timeout
        App-->>Customer: thông báo hủy đơn do nhà cung cấp không phản hồi, hoàn tiền tự động
    end
```
