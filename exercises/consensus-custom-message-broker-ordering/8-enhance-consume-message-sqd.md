# Enhance sequence — Consumer commit offset vào store độc lập với broker leader

Đây là **enhance** của flow đã có ở base (`3-base-consume-message-sqd.md`). So với base (offset lưu ngay trên broker duy nhất phụ trách partition), nay `CONSUMER_OFFSET_STORE` là 1 storage độc lập, tách rời khỏi broker leader hiện tại — khi leader đổi, consumer reconnect vào broker mới vẫn đọc đúng từ vị trí đã dừng, không đọc thiếu hoặc đọc trùng không kiểm soát — đáp ứng đúng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor Consumer
    participant OldLeader as Broker Leader cũ (partition P1)
    participant OffsetStore as CONSUMER_OFFSET_STORE (độc lập, không nằm trên broker nào)
    participant NewLeader as Broker Leader mới (sau failover)

    Consumer->>OldLeader: Fetch message từ offset=500
    OldLeader-->>Consumer: Trả message offset 500-520
    Consumer->>Consumer: Xử lý xong batch
    Consumer->>OffsetStore: Commit committed_offset=520 (ghi độc lập, không qua broker)
    OffsetStore-->>Consumer: Ack đã lưu

    Note over OldLeader: Broker Leader cũ crash ngay sau đó, cluster bầu NewLeader

    Consumer->>NewLeader: Reconnect, hỏi offset đã commit gần nhất
    NewLeader->>OffsetStore: Đọc committed_offset của consumer group cho partition P1
    OffsetStore-->>NewLeader: committed_offset=520
    NewLeader-->>Consumer: Tiếp tục fetch từ offset=521

    Note over Consumer,OffsetStore: Vì offset commit không gắn với broker leader nào, việc đổi leader không làm consumer đọc thiếu (bỏ sót 521 trở đi) hay đọc trùng (đọc lại từ 500)
```
