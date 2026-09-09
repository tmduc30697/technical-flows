# Base sequence — Read session

Đây là **base**, flow "Đọc session/trạng thái checkout" — dùng eventual consistency, phù hợp cho phần lớn truy cập chỉ đọc. Flow này bản thân không có vấn đề, nhưng nền tảng cho enhance vì cùng cơ chế này đang bị áp dụng luôn cho cả bước thanh toán nhạy cảm.

```mermaid
sequenceDiagram
    actor User
    participant App as Checkout App
    participant Nodes as SESSION_REPLICA (node bất kỳ)

    User->>App: Xem trạng thái giỏ hàng/checkout
    App->>Nodes: Đọc từ node gần nhất bất kỳ
    Nodes-->>App: Trả status (có thể trễ vài trăm ms, chấp nhận được)
    App-->>User: Hiển thị trạng thái
```
