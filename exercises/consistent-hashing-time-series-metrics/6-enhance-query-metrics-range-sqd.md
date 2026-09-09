# Enhance sequence — Query metrics theo khoảng thời gian (chỉ chạm partition tối thiểu)

Đây là **enhance** của flow `query-metrics-range` đã có ở base. So với base (phải scan toàn bộ dữ liệu trong mỗi time bucket rồi lọc), nay router tính được `hash(service_id)` để xác định chính xác `hash_range`, kết hợp với các `time_bucket` liên quan, nên chỉ cần chạm đúng các partition chứa dữ liệu của service đó, không đọc dư dữ liệu service khác. Đáp ứng yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor App as Dashboard App
    participant Router as Query Router
    participant PA1 as PARTITION (hash_range=A, 09:00-10:00)
    participant PA2 as PARTITION (hash_range=A, 10:00-11:00)
    participant PB as PARTITION (hash_range=B, ...)

    App->>Router: Lấy metrics 1 giờ qua của Service A
    Router->>Router: Tính hash_range=A từ service_id, xác định time_bucket 09:00-10:00 và 10:00-11:00
    Router->>PA1: Đọc trực tiếp partition (hash_range=A, 09:00-10:00)
    PA1-->>Router: Chỉ dữ liệu Service A
    Router->>PA2: Đọc trực tiếp partition (hash_range=A, 10:00-11:00)
    PA2-->>Router: Chỉ dữ liệu Service A
    Note over Router,PB: Partition B không hề bị chạm tới vì hash_range không khớp, tránh scan dư
    Router-->>App: Kết quả (không cần lọc lại vì partition đã tách theo service)
```
