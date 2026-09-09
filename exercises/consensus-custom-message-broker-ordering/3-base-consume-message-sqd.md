# Base sequence — Consumer đọc message và commit offset

Đây là **base**, flow "Consumer group đọc message từ 1 partition" ở trạng thái hiện tại — offset đã đọc tới đâu (`CONSUMER_OFFSET`) được lưu ngay trên broker duy nhất đang phụ trách partition, gắn chặt với broker đó. Flow này là tiền đề cho enhance vì yêu cầu 4 của đề bài chỉ ra rõ đây là vấn đề khi broker đổi.

```mermaid
sequenceDiagram
    actor Consumer
    participant Broker as Broker (duy nhất phụ trách partition P1)

    Consumer->>Broker: Fetch message từ offset=500
    Broker-->>Consumer: Trả message offset 500-520
    Consumer->>Consumer: Xử lý xong batch message
    Consumer->>Broker: Commit committed_offset=520
    Broker->>Broker: Lưu CONSUMER_OFFSET ngay trên chính broker này

    Note over Broker,Consumer: Nếu broker này chết, offset đã commit gắn liền với nó cũng biến mất hoặc không rõ trạng thái mới nhất
```
