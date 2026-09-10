# Sequence Diagram — Base: Request And Complete Trip

Đây là **base**, flow vòng đời chuyến đi cơ bản: tìm tài xế → nhận cuốc → đón khách → hoàn thành → thanh toán — tiền đề cho saga. Ở base, flow gọi tuần tự đơn giản, chưa xử lý mất kết nối, hủy giữa chừng, race condition, hay resume khi service crash.

```mermaid
sequenceDiagram
    actor Rider
    actor Driver
    participant Trip as Trip Service
    participant Matching as Matching Service
    participant Payment as Payment Service

    Rider->>Trip: Request trip
    Trip->>Trip: Create TRIP (status=requested)
    Trip->>Matching: Find nearby driver

    Matching-->>Trip: Driver found
    Trip->>Trip: Update TRIP (status=matched, driver_id)
    Trip-->>Driver: Trip offer
    Driver->>Trip: Accept trip

    Driver->>Trip: Mark picked up
    Trip->>Trip: Update TRIP (status=picked_up, picked_up_at)

    Driver->>Trip: Mark trip completed
    Trip->>Trip: Update TRIP (status=completed, completed_at)

    Trip->>Payment: Charge rider for trip
    Payment-->>Trip: Payment captured
    Trip->>Trip: Create PAYMENT_CHARGE (status=succeeded)

    Trip-->>Rider: Trip completed, receipt
    Note over Trip: No handling yet for disconnects, cancellations mid-flow, or service restarts
```
