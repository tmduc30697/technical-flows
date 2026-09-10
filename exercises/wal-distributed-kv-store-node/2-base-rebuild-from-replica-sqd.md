# Sequence Diagram — Base: Rebuild From Replica

Đây là **base**, flow phục hồi node duy nhất bằng cách rebuild toàn bộ từ replica qua network — cách duy nhất hiện có khi chưa có WAL cục bộ, tốn thời gian/network hơn hẳn và là tiền đề để so sánh với recovery nhanh bằng WAL cục bộ ở enhance.

```mermaid
sequenceDiagram
    participant Node as Cluster Node (crashed, restarted)
    participant Cluster as Cluster Coordinator
    participant Replica as Replica Node khác

    Node->>Node: Restart sau crash, dữ liệu cục bộ mất/không tin cậy
    Node->>Cluster: Thông báo cần rebuild, chưa sẵn sàng nhận request

    Cluster->>Replica: Chỉ định replica nguồn cho node đang rebuild
    Node->>Replica: Request toàn bộ KEY_VALUE_RECORD hiện có
    Replica-->>Node: Truyền toàn bộ dữ liệu qua network

    Node->>Node: Ghi lại toàn bộ dữ liệu nhận được vào local store
    Node->>Cluster: Rebuild hoàn tất, sẵn sàng rejoin
    Cluster-->>Node: Xác nhận rejoin cluster

    Note over Node,Replica: Quá trình này tốn network và thời gian đáng kể so với chỉ replay log cục bộ
```
