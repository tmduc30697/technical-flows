# Enhance sequence — Payment callback

Đây là **enhance**, flow "Nhận callback kết quả thanh toán từ đối tác". So với base, khi callback đến (dù trễ 20 giờ hay hơn), hệ thống không tra tỷ giá hiện tại nữa mà đọc lại `EXCHANGE_RATE_SNAPSHOT` đã khóa lúc buyer thanh toán để tính `seller_amount` cuối cùng — loại bỏ hoàn toàn rủi ro lệch số tiền do tỷ giá biến động trong lúc chờ.

```mermaid
sequenceDiagram
    participant PaymentPartner
    participant OrderService
    participant DB as Database

    Note over PaymentPartner,OrderService: Callback báo thành công đến sau 20 giờ, trong lúc đó tỷ giá thị trường đã biến động 2%
    PaymentPartner->>OrderService: POST /webhooks/payment { partner_ref, status=success }
    OrderService->>DB: Tìm PAYMENT_TRANSACTION theo partner_ref, lấy order_id
    OrderService->>DB: Lấy EXCHANGE_RATE_SNAPSHOT đã khóa của order này
    DB-->>OrderService: rate đã khóa lúc buyer thanh toán, không phải rate hiện tại
    OrderService->>OrderService: Tính lại seller_amount bằng đúng rate đã khóa
    OrderService->>DB: Cập nhật ORDER status=paid, ghi nhận seller_amount theo rate đã khóa
    OrderService->>DB: Cập nhật PAYMENT_TRANSACTION status=success
    Note over OrderService,DB: Test xác nhận seller_amount khớp với rate lúc khóa, không lệch 2% dù callback đến trễ 20 giờ
```
