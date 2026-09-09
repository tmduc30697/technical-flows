# Base sequence — Node failover (async replicate, không có log)

Đây là **base**, flow khi node authoritative của 1 trận đấu chết đột ngột. Do base chỉ replicate trạng thái bất đồng bộ (không có log sự kiện có thứ tự, không xác nhận majority trước khi commit), việc failover dễ làm mất input chưa kịp replicate hoặc áp dụng lại (double apply) input mà client gửi lại do timeout — đây là vấn đề mà yêu cầu 3 của đề bài yêu cầu enhance phải xử lý rõ ràng.

```mermaid
sequenceDiagram
    actor Client
    participant NodeA as Authoritative NODE (sắp chết)
    participant Replica as MATCH_STATE_REPLICA (follower, cũ hơn)
    participant NodeB as Follower NODE (tiếp quản)

    NodeA->>Replica: Replicate MATCH_STATE bất đồng bộ (có độ trễ)
    Client->>NodeA: Gửi INPUT_EVENT mới (ngay trước khi node chết)
    Note over NodeA: NodeA chết đột ngột trước khi kịp replicate input vừa nhận
    NodeA--xReplica: Không kịp replicate input mới nhất

    Note over NodeB: Cluster phát hiện NodeA chết, chọn NodeB tiếp quản dựa trên MATCH_STATE_REPLICA cũ nhất có sẵn
    NodeB->>NodeB: Nạp MATCH_STATE_REPLICA (đã cũ hơn thời điểm chết một khoảng)
    Note over NodeB: Input vừa gửi bị mất hẳn, người chơi cảm giác thao tác bị "nuốt"

    Client->>NodeB: Client timeout, tự động gửi lại input tương tự
    NodeB->>NodeB: Áp dụng input gửi lại như một input mới hoàn toàn
    Note over NodeB: Không có cách phân biệt input này đã từng được xử lý một phần ở NodeA hay chưa, rủi ro áp dụng trùng
```
