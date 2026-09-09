# Base sequence — Checkout payment

Đây là **base**, flow "Buyer bấm thanh toán" — chọn flow này vì đây chính là nơi tỷ giá lẽ ra phải được khóa lại nhưng ở base chưa làm vậy. Hệ thống chỉ tra tỷ giá hiện tại (live) để tính số tiền seller nhận tại thời điểm này, không lưu lại tỷ giá đã dùng gắn với order.

```mermaid
sequenceDiagram
    actor Buyer
    participant WebApp
    participant OrderService
    participant RateService as Exchange Rate Service (live)
    participant DB as Database
    participant PaymentPartner

    Buyer->>WebApp: Bấm Thanh toán
    WebApp->>OrderService: POST /checkout
    OrderService->>RateService: Lấy tỷ giá hiện tại cho currency_pair
    RateService-->>OrderService: rate hiện tại
    OrderService->>OrderService: Tính seller_amount = buyer_amount x rate
    OrderService->>DB: Tạo ORDER (buyer_amount, seller_amount, status=pending)
    Note over OrderService,DB: Không lưu lại rate đã dùng, chỉ lưu seller_amount đã tính sẵn
    OrderService->>PaymentPartner: Khởi tạo phiên thanh toán
    PaymentPartner-->>OrderService: payment_partner_ref
    OrderService->>DB: Tạo PAYMENT_TRANSACTION (order_id, partner_ref, status=pending)
    OrderService-->>WebApp: Redirect tới trang thanh toán của partner
    WebApp-->>Buyer: Chuyển tới cổng thanh toán
```
