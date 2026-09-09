# Enhance sequence — Partition write accept & merge

Đây là **enhance**, cùng flow "Agent write during partition" đã có ở base nhưng nay thay đổi hoàn toàn theo yêu cầu thứ 3 của đề bài: ưu tiên availability — chấp nhận ghi phía minority, và merge theo timestamp agent gửi khi partition hàn lại thay vì từ chối như base.

```mermaid
sequenceDiagram
    actor Agent as Agent trên server (phía minority)
    participant NodesMinority as TIME_SERIES_NODE (nhóm minority)
    participant Policy as PARTITION_WRITE_POLICY store
    participant NodesMajority as TIME_SERIES_NODE (nhóm majority)

    Note over NodesMinority: Đang xảy ra network partition
    Agent->>NodesMinority: Gửi METRIC_SAMPLE (sample_timestamp=t1)
    NodesMinority->>Policy: Kiểm tra accept_minority_writes
    Policy-->>NodesMinority: true — ưu tiên availability cho dữ liệu giám sát
    NodesMinority->>NodesMinority: Ghi nhận cục bộ, không chờ majority
    NodesMinority-->>Agent: Ghi thành công (tạm thời, phía minority)

    Note over NodesMinority,NodesMajority: Cùng lúc, phía majority cũng nhận ghi metric cho cùng server (nếu agent gửi trùng qua đường khác) hoặc dữ liệu khác thời điểm

    Note over NodesMinority,NodesMajority: Partition hàn lại
    NodesMinority->>NodesMajority: Đồng bộ lại toàn bộ METRIC_SAMPLE đã ghi trong lúc partition
    NodesMajority->>Policy: Áp dụng conflict_resolution=by_agent_send_timestamp
    NodesMajority->>NodesMajority: Với các bản ghi trùng lấn, giữ theo sample_timestamp agent gửi (không theo received_at của từng node)
    NodesMajority-->>Agent: Dữ liệu giám sát của giai đoạn partition được giữ đầy đủ, không mất mát như base
```
