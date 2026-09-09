# Enhance sequence — Follow during partition

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có quyết định rõ ràng cho partition). Đáp ứng yêu cầu thứ 4 của đề bài: ưu tiên availability cho hành động follow, chấp nhận ghi phía minority, merge theo "follow thắng nếu trùng thời điểm không xác định được" khi partition hàn lại.

```mermaid
sequenceDiagram
    actor User
    participant NodesMinority as RELATIONSHIP_REPLICA (phía minority, đang partition)
    participant Policy as PARTITION_WRITE_POLICY store
    participant NodesMajority as RELATIONSHIP_REPLICA (phía majority)

    Note over NodesMinority: Đang xảy ra network partition
    User->>NodesMinority: Bấm Follow
    NodesMinority->>Policy: Kiểm tra PARTITION_WRITE_POLICY(action=follow)
    Policy-->>NodesMinority: accept_minority_writes=true
    NodesMinority->>NodesMinority: Ghi status=following cục bộ, không chờ majority
    NodesMinority-->>User: "Đã follow" — trải nghiệm không bị gián đoạn

    Note over NodesMinority,NodesMajority: Partition hàn lại, phát hiện 2 phía có trạng thái khác nhau cho cùng cặp quan hệ và không xác định được thứ tự thời gian chính xác (do đồng hồ lệch giữa 2 phía)
    NodesMinority->>NodesMajority: Đồng bộ trạng thái đã ghi trong lúc partition
    NodesMajority->>Policy: Áp dụng conflict_resolution=follow_wins_if_ambiguous
    NodesMajority->>NodesMajority: Nếu không xác định rõ ai xảy ra sau, ưu tiên giữ status=following
    Note over NodesMajority: Chấp nhận thiên về "false positive follow" hơn là làm mất thao tác follow của user trong lúc sự cố mạng
```
