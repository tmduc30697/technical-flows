# Sequence Diagram — Base: Produce and Consume Message

Đây là **base**, flow gửi/nhận message chỉ dựa vào bộ nhớ (không WAL) — tiền đề để thấy rõ rủi ro mất message khi broker crash, chính là lý do đề bài yêu cầu thêm WAL.

```mermaid
sequenceDiagram
    actor Producer
    participant Broker as Broker (in-memory queue)
    actor Consumer

    Producer->>Broker: Send message
    Broker->>Broker: Lưu MESSAGE vào bộ nhớ (ack_status = pending)
    Broker-->>Producer: Ack "đã nhận"

    Consumer->>Broker: Poll message
    Broker-->>Consumer: Trả message
    Consumer->>Consumer: Xử lý message
    Consumer->>Broker: Ack đã xử lý xong
    Broker->>Broker: Set ack_status = acked (chỉ trong bộ nhớ)

    Note over Broker: Nếu broker crash bất kỳ lúc nào trước khi consumer kịp đọc/ack, message mất hoàn toàn dù producer đã nhận ack "đã nhận"
```
