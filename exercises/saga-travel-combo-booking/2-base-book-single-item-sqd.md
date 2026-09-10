# Sequence Diagram — Base: Book Single Item

Đây là **base**, flow đặt riêng lẻ một phần (ví dụ đặt khách sạn) qua service nhà cung cấp tương ứng — tiền đề cho saga combo: đây là building block mà saga combo sau này sẽ gọi lặp lại cho cả 3 nhà cung cấp. Ở base, mỗi lần đặt độc lập, chưa có khái niệm phối hợp nhiều phần cùng lúc hay hủy dây chuyền.

```mermaid
sequenceDiagram
    actor Customer
    participant Booking as Booking Service
    participant Hotel as Hotel Provider (Partner)

    Customer->>Booking: Book a hotel room (hotel_id, dates)
    Booking->>Hotel: Create reservation request
    Hotel-->>Booking: Reservation confirmed

    Booking->>Booking: Create HOTEL_BOOKING (status=confirmed)
    Booking-->>Customer: Hotel booking confirmed

    Note over Booking,Hotel: This booking exists on its own, unaware of any flight or car booking
```
