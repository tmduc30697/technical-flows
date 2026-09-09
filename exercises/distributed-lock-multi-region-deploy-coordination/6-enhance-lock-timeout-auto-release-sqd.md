# Sequence Diagram - Enhance: lock-timeout-auto-release

Đây là flow **enhance mới**, chưa tồn tại ở base: khi pipeline giữ lock bị treo (CI lỗi, không renew lease) vượt quá `lease_ttl_seconds`, coordinator tự động hết hạn và nhả lock, đồng thời gửi cảnh báo cho người vận hành thay vì chặn các pipeline khác vô thời hạn. Đáp ứng yêu cầu 2.

```mermaid
sequenceDiagram
    participant CDA as Pipeline Region A (đang giữ lock, bị treo)
    participant Coord as Lock Coordinator ensemble (ZK/etcd)
    participant Audit as Audit Log
    participant Alert as Alerting/Ops
    participant CDB as Pipeline Region B (đang chờ)

    CDA->>Coord: Acquire lock thành công, lease_ttl = 5 phút
    Note over CDA: CI job bị treo do lỗi runner, không renew lease, không release lock
    Coord->>Coord: Phát hiện lease hết hạn (quá lease_ttl_seconds)
    Coord->>Coord: Tự động expire và release lock
    Coord->>Audit: Ghi log LOCK_EXPIRED (region=A, lý do=lease timeout)
    Coord->>Alert: Gửi cảnh báo LOCK_STUCK cho người vận hành
    Alert-->>Alert: Người vận hành kiểm tra pipeline Region A bị treo
    Coord->>CDB: Cấp lock cho pipeline đang chờ tiếp theo (Region B)
    CDB->>Coord: Xác nhận acquire, tiếp tục deploy Region B bình thường
```
