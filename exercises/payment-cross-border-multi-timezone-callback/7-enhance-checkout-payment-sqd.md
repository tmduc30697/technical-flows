# Enhance sequence — Checkout payment

Đây là **enhance**, flow "Buyer bấm thanh toán". So với base, ngay tại thời điểm này hệ thống không chỉ tính `seller_amount` rồi bỏ qua tỷ giá đã dùng, mà còn lưu lại `EXCHANGE_RATE_SNAPSHOT` gắn với order — khóa đúng tỷ giá tại thời điểm giao dịch để mọi xử lý callback sau này (dù trễ bao lâu) đều dùng lại đúng con số này.

```mermaid
sequenceDiagram
    actor Buyer
    participant WebApp
    participant OrderService
    participant RateService as Exchange Rate Service
    participant DB as Database
    participant PaymentPartner

    Buyer->>WebApp: Bấm Thanh toán
    WebApp->>OrderService: POST /checkout
    OrderService->>RateService: Lấy tỷ giá hiện tại cho currency_pair
    RateService-->>OrderService: rate
    OrderService->>OrderService: Tính seller_amount = buyer_amount x rate
    OrderService->>DB: Tạo ORDER (buyer_amount, seller_amount, status=pending)
    OrderService->>DB: Tạo EXCHANGE_RATE_SNAPSHOT (order_id, rate, locked_at=now)
    Note over OrderService,DB: Tỷ giá được khóa và lưu lại ngay tại đây, không tính lại ở bất kỳ bước nào sau này
    OrderService->>PaymentPartner: Khởi tạo phiên thanh toán
    PaymentPartner-->>OrderService: payment_partner_ref
    OrderService->>DB: Tạo PAYMENT_TRANSACTION (order_id, partner_ref, status=pending)
    OrderService-->>WebApp: Redirect tới trang thanh toán của partner
    WebApp-->>Buyer: Chuyển tới cổng thanh toán
```
