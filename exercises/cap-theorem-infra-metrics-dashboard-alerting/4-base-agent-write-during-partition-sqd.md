# Base sequence — Agent write during partition (minority bị từ chối, mất dữ liệu)

Đây là **base**, flow "Agent gửi metrics lúc network partition" ở trạng thái hiện tại — ghi luôn yêu cầu đạt write_quorum chung, nên phía minority bị từ chối hoàn toàn. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 3 của đề bài chính là quyết định lại cách xử lý ở đây.

```mermaid
sequenceDiagram
    actor Agent as Agent trên server (phía minority)
    participant NodesMinority as TIME_SERIES_NODE (nhóm minority)
    participant Config as QUORUM_CONFIG store

    Note over NodesMinority: Đang xảy ra network partition, nhóm này không đạt được majority
    Agent->>NodesMinority: Gửi METRIC_SAMPLE (cpu/memory/disk)
    NodesMinority->>Config: Kiểm tra write_quorum chung
    NodesMinority-->>Agent: Từ chối ghi vì không đạt write_quorum
    Note over Agent,NodesMinority: Agent tiếp tục thử gửi nhưng liên tục bị từ chối trong suốt thời gian partition — toàn bộ dữ liệu giám sát của các server phía minority bị mất, không có bản ghi nào phục vụ điều tra sau này
```
