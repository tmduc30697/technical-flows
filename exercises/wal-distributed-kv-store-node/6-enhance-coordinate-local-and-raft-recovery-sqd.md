# Sequence Diagram — Enhance: Coordinate Local and Raft Recovery

Đây là **enhance**, flow mới xử lý yêu cầu phối hợp rõ ràng giữa recovery cục bộ (WAL) và recovery ở tầng cluster (Raft log) — không để 2 cơ chế mâu thuẫn nhau về thứ tự áp dụng thay đổi, đặc biệt quan trọng khi nhiều node cùng crash đồng thời (mất điện datacenter).

```mermaid
sequenceDiagram
    participant Node as Cluster Node (crashed, restarted)
    participant WAL as Local WAL (disk)
    participant Cluster as Cluster Coordinator (Raft)

    Node->>WAL: Replay LOCAL_WAL_ENTRY, apply tới last_applied_lsn
    Node->>Node: Ghi nhận last_applied_lsn tương ứng với Raft index nào (nếu WAL có lưu ánh xạ)

    Node->>Cluster: Rejoin, hỏi Raft log từ last known raft_index trở đi
    Cluster-->>Node: Trả các RAFT_LOG_ENTRY còn thiếu kể từ điểm đó

    loop mỗi RAFT_LOG_ENTRY còn thiếu
        Node->>Node: Apply theo đúng thứ tự raft_index, chỉ áp dụng phần chưa có trong WAL cục bộ
    end

    Node->>Node: Cập nhật NODE_RECOVERY_STATE.last_applied_raft_index
    Note over Node,Cluster: WAL cục bộ phục hồi trạng thái riêng của node, Raft log đồng bộ phần cluster đã đồng thuận sau đó — không apply trùng hoặc sai thứ tự giữa 2 nguồn

    Node->>Cluster: Đồng thuận state chung hoàn tất
```
