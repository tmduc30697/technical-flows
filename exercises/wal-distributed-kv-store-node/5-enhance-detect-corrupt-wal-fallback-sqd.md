# Sequence Diagram — Enhance: Detect Corrupt WAL Fallback

Đây là **enhance**, flow mới xử lý yêu cầu: khi WAL cục bộ bị corrupt hoàn toàn (đĩa hỏng), node phải tự phát hiện và chuyển sang chế độ rebuild-from-replica (tái sử dụng flow base [2-base-rebuild-from-replica-sqd.md](2-base-rebuild-from-replica-sqd.md)) thay vì cố replay dữ liệu hỏng.

```mermaid
sequenceDiagram
    participant Node as Cluster Node (crashed, restarted)
    participant WAL as Local WAL (disk)
    participant Cluster as Cluster Coordinator
    participant Replica as Replica Node khác

    Node->>Node: Restart, set recovery_mode = replaying_local_wal
    Node->>WAL: Đọc LOCAL_WAL_ENTRY

    alt WAL đọc được và checksum hợp lệ
        Node->>Node: Replay bình thường (xem 4-enhance-node-local-wal-recovery-sqd.md)
    else WAL corrupt hoàn toàn (đĩa hỏng, không đọc được hoặc checksum sai hàng loạt)
        Node->>Node: Set recovery_mode = rebuild_from_replica
        Node->>Cluster: Báo WAL cục bộ hỏng, không dùng để phục hồi
        Cluster->>Replica: Chỉ định replica nguồn
        Node->>Replica: Rebuild toàn bộ dữ liệu (giống flow base)
        Replica-->>Node: Truyền dữ liệu
        Node->>Cluster: Rebuild hoàn tất, sẵn sàng rejoin
    end
```
