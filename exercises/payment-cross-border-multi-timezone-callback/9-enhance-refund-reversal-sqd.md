# Enhance sequence — Refund reversal

Đây là **enhance**, flow hoàn toàn mới so với base — xử lý đúng trường hợp callback chính thức báo thất bại sau khi order đã bị polling dự phòng (xem `4-base-fallback-status-polling-sqd.md`) chuyển tạm sang "đã thanh toán". Hệ thống phải hủy đơn và hoàn tiền theo đúng tỷ giá đã khóa ban đầu, không phải tỷ giá hiện tại lúc hoàn tiền.

```mermaid
sequenceDiagram
    participant PaymentPartner
    participant OrderService
    participant DB as Database

    Note over OrderService,DB: Trước đó, luồng polling dự phòng đã tạm chuyển ORDER sang status=paid do chưa nhận được callback
    PaymentPartner->>OrderService: POST /webhooks/payment { partner_ref, status=failed } (callback chính thức đến trễ)
    OrderService->>DB: Tìm ORDER theo partner_ref, phát hiện status hiện tại đang là paid do polling
    Note over OrderService: Phát hiện mâu thuẫn, callback chính thức báo thất bại
    OrderService->>DB: Lấy EXCHANGE_RATE_SNAPSHOT đã khóa của order này
    DB-->>OrderService: rate đã khóa lúc buyer thanh toán
    OrderService->>DB: Cập nhật ORDER status=refunded
    OrderService->>DB: Tạo REFUND (order_id, amount tính theo rate đã khóa, reason=late_callback_failed)
    OrderService->>PaymentPartner: Yêu cầu hoàn tiền cho buyer
    PaymentPartner-->>OrderService: Xác nhận đã hoàn tiền
    Note over OrderService,DB: Số tiền hoàn luôn tính theo rate đã khóa ban đầu, không dùng rate hiện tại tại thời điểm hoàn tiền
```
