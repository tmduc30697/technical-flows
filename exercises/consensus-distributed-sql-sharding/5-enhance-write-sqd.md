# Enhance sequence — Ghi dữ liệu đúng leader của range (multi-raft + redirect)

Đây là **enhance** của flow đã có ở base (`2-base-write-sqd.md`). So với base (chỉ 1 range nên không cần định tuyến), nay bảng chia thành nhiều range, mỗi range có `RAFT_GROUP` riêng với leader có thể ở node khác nhau — client phải luôn được route tới đúng leader hiện tại, nếu gửi nhầm follower sẽ nhận lại redirect hoặc lỗi NotLeader — đáp ứng đúng yêu cầu 1 và yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor Client
    participant Cache as LEADER_ROUTING_CACHE (client-side)
    participant Follower as Node X (follower của Range B, cache đã cũ)
    participant LeaderB as Node Y (leader thực sự của Range B)

    Client->>Cache: Tra leader hiện tại của Range B (key=42 thuộc range B)
    Cache-->>Client: cached_leader_node_id = Node X (đã cũ, Range B vừa đổi leader)

    Client->>Follower: Write row(key=42, value)
    Follower->>Follower: Kiểm tra role, không phải leader của Range B
    Follower-->>Client: NotLeader, kèm known_leader = Node Y (nếu đã biết)

    Client->>Cache: Cập nhật cached_leader_node_id = Node Y
    Client->>LeaderB: Write row(key=42, value) (retry đúng leader)
    LeaderB->>LeaderB: Append log entry, replicate tới majority Range B
    LeaderB-->>Client: 200 OK

    Note over Client,LeaderB: Leader của Range A (không liên quan) có thể đang nằm ở 1 node hoàn toàn khác — mỗi range định tuyến độc lập
```
