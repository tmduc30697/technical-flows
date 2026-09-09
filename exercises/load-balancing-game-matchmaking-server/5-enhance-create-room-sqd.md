# Enhance sequence — Create room (consistent hashing + lease TTL + cân bằng theo số room)

Đây là **enhance** của flow `create-room` đã có ở base. So với base (round-robin theo connection), giờ đây: (1) vị trí room trên hash ring được tính từ `room_id` (yêu cầu 1), (2) server được chọn ưu tiên theo `active_room_count` thấp nhất trong nhóm ứng viên gần nhất trên ring, không phải theo connection count (yêu cầu 4), và (3) server phải giữ được `ROOM_LEASE` có TTL trước khi coi là chủ sở hữu room, đặt nền cho chống split-brain (yêu cầu 3).

```mermaid
sequenceDiagram
    actor Player1 as Player 1
    participant LB as Load Balancer / Hash Ring
    participant SvcB as SERVER_INSTANCE B
    participant LeaseStore as ROOM_LEASE store
    participant DB as Room Registry

    Player1->>LB: Tạo room mới (room_id = R1)
    LB->>LB: Băm room_id ra room_hash_slot trên consistent hash ring
    LB->>LB: Trong các server gần slot đó, chọn server có active_room_count thấp nhất
    LB->>SvcB: Route yêu cầu tạo room R1 tới SERVER_INSTANCE B
    SvcB->>LeaseStore: Acquire ROOM_LEASE(room_id=R1, server=B, TTL=10s)
    LeaseStore-->>SvcB: Lease granted, lease_token=T1
    SvcB->>DB: Ghi ROOM(id=R1, room_hash_slot, status=active)
    DB-->>SvcB: OK
    SvcB->>SvcB: Bắt đầu vòng lặp renew lease T1 trước khi hết TTL
    SvcB-->>Player1: Room được tạo, đang kết nối vào server B
    Note over LB,LeaseStore: Vị trí trên hash ring và lease đảm bảo mọi request liên quan R1 sau này luôn resolve về đúng server B
```
