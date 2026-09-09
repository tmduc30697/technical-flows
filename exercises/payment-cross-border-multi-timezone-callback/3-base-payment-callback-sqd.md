# Base sequence — Payment callback

Đây là **base**, flow "Nhận callback kết quả thanh toán từ đối tác" — chọn flow này vì đây chính là nơi bug xảy ra: callback có thể đến trễ (hàng chục giờ), nhưng hệ thống base lại tra tỷ giá hiện tại **lần nữa** tại thời điểm callback đến để tính số tiền cuối cùng, thay vì dùng đúng tỷ giá lúc buyer thanh toán.

```mermaid
sequenceDiagram
    participant PaymentPartner
    participant OrderService
    participant RateService as Exchange Rate Service (live)
    participant DB as Database

    PaymentPartner->>OrderService: POST /webhooks/payment { partner_ref, status=success } (có thể đến trễ)
    OrderService->>DB: Tìm PAYMENT_TRANSACTION theo partner_ref, lấy order_id
    OrderService->>RateService: Lấy tỷ giá hiện tại cho currency_pair
    RateService-->>OrderService: rate hiện tại (đã khác lúc buyer thanh toán nếu callback đến trễ)
    OrderService->>OrderService: Tính lại seller_amount theo rate hiện tại này
    OrderService->>DB: Cập nhật ORDER status=paid, ghi nhận seller_amount vừa tính lại
    OrderService->>DB: Cập nhật PAYMENT_TRANSACTION status=success
    Note over OrderService,RateService: Nếu tỷ giá đã biến động trong lúc chờ callback, số tiền seller nhận sẽ bị lệch so với lúc buyer thanh toán
```
