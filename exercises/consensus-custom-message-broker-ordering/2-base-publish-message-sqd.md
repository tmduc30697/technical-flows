# Base sequence — Publish message (ack ngay khi broker duy nhất nhận được)

Đây là **base**, flow "Producer publish message vào 1 partition" ở trạng thái hiện tại — broker duy nhất phụ trách partition gán offset và ack cho producer ngay khi nhận được, không có replica nào xác nhận. Flow này là tiền đề cho enhance vì yêu cầu 1 của đề bài chính là thay đổi thời điểm ack này.

```mermaid
sequenceDiagram
    actor Producer
    participant Broker as Broker (duy nhất phụ trách partition P1)

    Producer->>Broker: Publish message vào partition P1
    Broker->>Broker: Gán offset=1000, lưu vào local log
    Broker-->>Producer: Ack "publish thành công", offset=1000

    Note over Broker: Ack ngay khi ghi local, chưa có node nào khác xác nhận đã nhận được message này
```
