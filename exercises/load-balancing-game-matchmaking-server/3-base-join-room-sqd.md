# Base sequence — Join room (misrouting vì LB route độc lập từng request)

Đây là **base**, flow người chơi thứ 2 join vào room đã tồn tại. Vì load balancer route độc lập từng request (không biết room đã được gán cho server nào), player join sau có thể bị route sai server, không thấy được state của room. Đây chính là vấn đề gốc mà yêu cầu 1 của đề bài (consistent hashing theo `room_id`) phải giải quyết.

```mermaid
sequenceDiagram
    actor Player2 as Player 2
    participant LB as Load Balancer
    participant SvcA as SERVER_INSTANCE A
    participant SvcB as SERVER_INSTANCE B (đang giữ room)
    participant DB as Room Registry

    Player2->>LB: Join room (room_id = R1)
    LB->>LB: Chọn server theo round-robin hiện tại (không tra cứu room_id)
    LB->>SvcA: Route request join room R1 tới SERVER_INSTANCE A
    SvcA->>DB: Tìm room R1 trong dữ liệu cục bộ của A
    DB-->>SvcA: Không tìm thấy room R1 (room thực tế đang ở server B)
    SvcA-->>Player2: Lỗi "room not found" hoặc trạng thái game trống, không đồng bộ với Player 1
    Note over LB,SvcA: Player 2 bị route sai server, không thể chơi cùng Player 1 dù cùng room_id
```
