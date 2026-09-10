# Sequence Diagram — Enhance: Checkpoint Truncate WAL

Đây là **enhance**, flow mới xử lý yêu cầu checkpoint định kỳ cắt bớt phần WAL chứa toàn message đã được mọi consumer liên quan ack xong, tránh WAL phình vô hạn khiến recovery time ngày càng dài khi nhiều producer ghi đồng thời.

```mermaid
sequenceDiagram
    participant Scheduler as Checkpoint Scheduler
    participant Broker as Broker
    participant WAL as WAL (disk)

    Scheduler->>Broker: Trigger checkpoint định kỳ
    Broker->>WAL: Tìm lsn lớn nhất mà mọi MESSAGE trước đó đều đã acked
    WAL-->>Broker: up_to_lsn

    Broker->>Broker: Tạo CHECKPOINT (up_to_lsn)
    Broker->>WAL: Cắt/archive các WAL_ENTRY có lsn nhỏ hơn up_to_lsn
    WAL-->>Broker: Đã cắt xong

    Note over Broker,WAL: Recovery lần sau chỉ cần replay từ up_to_lsn trở đi, không phải toàn bộ lịch sử
    Broker-->>Scheduler: Checkpoint hoàn tất
```
