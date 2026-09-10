# Sequence Diagram — Enhance: Produce and Consume Message

Đây là **enhance**, flow đã thay đổi so với base ([2-base-produce-consume-message-sqd.md](2-base-produce-consume-message-sqd.md)): broker chỉ ack producer sau khi WAL fsync thành công, và mỗi bước giao message cho consumer / consumer ack đều ghi `WAL_ENTRY` riêng, thay vì chỉ tồn tại trong bộ nhớ như base.

```mermaid
sequenceDiagram
    actor Producer
    participant Broker as Broker
    participant WAL as WAL (disk)
    actor Consumer

    Producer->>Broker: Send message
    Broker->>WAL: Append WAL_ENTRY (event_type = received) + fsync

    alt fsync thành công
        WAL-->>Broker: Durable
        Broker-->>Producer: Ack "đã nhận"
    else broker crash giữa lúc ghi WAL, trước fsync
        Note over Broker,WAL: Không trả ack, producer coi như gửi thất bại và tự retry
    end

    Consumer->>Broker: Poll message
    Broker->>WAL: Append WAL_ENTRY (event_type = delivered, consumer_id)
    Broker-->>Consumer: Trả message

    Consumer->>Consumer: Xử lý message
    Consumer->>Broker: Ack đã xử lý xong
    Broker->>WAL: Append WAL_ENTRY (event_type = acked) + fsync
    Broker->>Broker: Set MESSAGE.ack_status = acked
```
