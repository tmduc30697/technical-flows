# Enhance sequence — Join room (route đúng server nhờ consistent hashing)

Đây là **enhance** của flow `join-room` đã có ở base. So với base (Player 2 bị route sai server vì LB không tra cứu room_id), giờ LB băm `room_id` ra đúng slot trên hash ring và route thẳng tới server đang giữ lease của room đó, đáp ứng trực tiếp yêu cầu 1 của đề bài — mọi player join sau đều tới đúng instance dù kết nối ở thời điểm khác nhau.

```mermaid
sequenceDiagram
    actor Player2 as Player 2
    participant LB as Load Balancer / Hash Ring
    participant SvcB as SERVER_INSTANCE B (đang giữ lease room R1)
    participant LeaseStore as ROOM_LEASE store

    Player2->>LB: Join room (room_id = R1)
    LB->>LB: Băm room_id = R1 ra cùng room_hash_slot đã tính lúc tạo room
    LB->>LeaseStore: Tra cứu server đang giữ lease active cho slot/room R1
    LeaseStore-->>LB: SERVER_INSTANCE B đang giữ lease active
    LB->>SvcB: Route request join room R1 tới SERVER_INSTANCE B
    SvcB-->>Player2: Join thành công, đồng bộ đúng state hiện tại với Player 1
    Note over LB,SvcB: Vì hash room_id ra cùng 1 slot mỗi lần, mọi player join sau đều tới đúng server đang thật sự giữ room
```
