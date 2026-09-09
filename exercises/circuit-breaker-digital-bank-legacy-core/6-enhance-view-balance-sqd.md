# Enhance sequence — View balance (fallback cache gắn nhãn rõ ràng)

Đây là **enhance**, cùng flow "View balance" đã có ở base nhưng nay thay đổi theo yêu cầu thứ 3 của đề bài: khi breaker mở, fallback về số dư cache gần nhất, nhưng hiển thị rõ đây không phải real-time.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Digital Banking App
    participant Breaker as CIRCUIT_BREAKER_STATE
    participant Core as Core Banking (legacy)
    participant Cache as BALANCE_CACHE store

    Customer->>App: Xem số dư tài khoản
    App->>Breaker: Kiểm tra state
    alt Breaker = closed
        App->>Core: Gọi lấy số dư real-time
        Core-->>App: Trả số dư
        App->>Cache: Cập nhật BALANCE_CACHE(cached_balance, cached_at=now)
        App-->>Customer: Hiển thị số dư real-time
    else Breaker = open (core đang bảo vệ, không gọi nữa)
        App->>Cache: Lấy cached_balance + cached_at gần nhất
        Cache-->>App: Trả số dư đã cache
        App-->>Customer: Hiển thị số dư kèm nhãn rõ "Số dư tại {cached_at}, không phải real-time"
        Note over App,Customer: Khách hàng biết rõ đây là dữ liệu trễ, không nhầm tưởng là số dư tức thời
    end
```
