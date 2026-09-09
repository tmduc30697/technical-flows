# Enhance sequence — Partition minority rejection

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đáp ứng yêu cầu thứ 3 của đề bài: khi có network partition, nhóm minority phải từ chối trừ kho để tránh vượt tồn kho thực khi partition hàn lại.

```mermaid
sequenceDiagram
    participant Detector as Partition Detector
    participant NodesA as Nhóm node A (majority)
    participant NodesB as Nhóm node B (minority)
    actor UserA as User được route tới nhóm A
    actor UserB as User được route tới nhóm B

    Detector->>NodesA: Kiểm tra khả năng liên lạc giữa các node
    Detector->>NodesB: Kiểm tra khả năng liên lạc giữa các node
    Detector->>Detector: Xác định NodesA đủ số node để đạt quorum (is_majority=true), NodesB không đủ (is_majority=false)
    Detector->>NodesA: Ghi PARTITION_STATUS(partition_group=A, is_majority=true)
    Detector->>NodesB: Ghi PARTITION_STATUS(partition_group=B, is_majority=false)

    UserA->>NodesA: Mua hàng, yêu cầu trừ kho
    NodesA->>NodesA: Đủ node để đạt write_quorum hiện tại → cho phép trừ kho
    NodesA-->>UserA: Mua hàng thành công

    UserB->>NodesB: Mua hàng, yêu cầu trừ kho
    NodesB->>NodesB: Kiểm tra PARTITION_STATUS.is_majority = false
    NodesB-->>UserB: Từ chối trừ kho — "Hệ thống đang bận, vui lòng thử lại sau"
    Note over NodesB: Nhóm minority không tự ý bán dù vẫn nhận được request, tránh vượt tồn kho thực khi 2 nhóm partition hàn lại
```
