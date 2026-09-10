# Sequence Diagram — Enhance: Node Local WAL Recovery

Đây là **enhance**, flow mới hoàn toàn — thay thế con đường mặc định của base ([2-base-rebuild-from-replica-sqd.md](2-base-rebuild-from-replica-sqd.md)): node tự phục hồi từ WAL cục bộ trước, chỉ báo "sẵn sàng" với cluster sau khi replay xong, nhanh hơn hẳn so với rebuild qua network.

```mermaid
sequenceDiagram
    participant Node as Cluster Node (crashed, restarted)
    participant WAL as Local WAL (disk)
    participant Cluster as Cluster Coordinator

    Node->>Node: Restart sau crash, set recovery_mode = replaying_local_wal
    Node->>WAL: Đọc LOCAL_WAL_ENTRY theo thứ tự lsn

    loop mỗi entry
        Node->>Node: Verify checksum
        Node->>Node: Apply vào KEY_VALUE_RECORD cục bộ
    end

    Node->>Node: Cập nhật NODE_RECOVERY_STATE.last_applied_lsn

    Node->>Cluster: Thông báo "sẵn sàng trở lại" (rejoin), chỉ sau khi replay hoàn tất
    Note over Node,Cluster: Không thông báo sẵn sàng khi còn đang replay, tránh trả dữ liệu cũ/thiếu

    Cluster-->>Node: Xác nhận rejoin, tiếp tục đồng bộ Raft log nếu cần (xem 6-enhance-coordinate-local-and-raft-recovery-sqd.md)
```
