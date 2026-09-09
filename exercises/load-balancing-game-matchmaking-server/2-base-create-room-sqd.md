# Base sequence — Create room (gán server bằng round-robin đơn giản)

Đây là **base**, flow tạo room ở trạng thái hiện tại: load balancer chọn server instance để giữ room bằng round-robin/random dựa trên số connection hiện có, không dùng consistent hashing theo `room_id`. Flow này là tiền đề để so sánh với yêu cầu 1 và yêu cầu 4 của đề bài (gán cố định qua hashing, cân bằng theo số room chứ không phải số connection).

```mermaid
sequenceDiagram
    actor Player1 as Player 1
    participant LB as Load Balancer
    participant SvcA as SERVER_INSTANCE A
    participant SvcB as SERVER_INSTANCE B
    participant DB as Room Registry

    Player1->>LB: Tạo room mới
    LB->>LB: Chọn server theo round-robin (dựa trên active_connection_count)
    LB->>SvcB: Route yêu cầu tạo room tới SERVER_INSTANCE B
    SvcB->>DB: Ghi ROOM(assigned_server_id=B, status=active)
    DB-->>SvcB: OK
    SvcB-->>Player1: Room được tạo, đang kết nối vào server B
    Note over LB,SvcB: Việc gán chỉ dựa vào số connection tại thời điểm tạo, không có cơ chế nào đảm bảo các request liên quan tới room này sau đó luôn tới đúng server B
```
