# Enhance sequence — Server crash giữa trận, failover an toàn nhờ lease TTL

Đây là **enhance**, flow hoàn toàn mới xử lý khi server đang giữ room bị crash giữa trận. Đáp ứng yêu cầu 2 (phát hiện qua health check, chiến lược rõ ràng là khôi phục từ snapshot gần nhất, không im lặng drop kết nối) và yêu cầu 3 (lease TTL hết hạn trước khi server khác được nhận quyền giữ room, tránh 2 server cùng tin mình giữ room do stale routing table).

```mermaid
sequenceDiagram
    participant HC as Health Checker
    participant SvcB as SERVER_INSTANCE B (đã crash)
    participant LeaseStore as ROOM_LEASE store
    participant SnapStore as ROOM_SNAPSHOT store
    participant SvcC as SERVER_INSTANCE C (server mới nhận room)
    participant LB as Load Balancer / Hash Ring
    actor Players as Players trong room R1

    HC->>SvcB: Health check định kỳ
    SvcB--xHC: Không phản hồi (đã crash)
    HC->>HC: Đánh dấu SERVER_INSTANCE B = down sau N lần check liên tiếp thất bại
    Note over LeaseStore: Lease của B cho room R1 không được renew, tự hết hạn theo TTL, không ai được gán quyền cho tới khi lease thật sự expired
    LeaseStore->>LeaseStore: Lease(room=R1, server=B) chuyển status=expired
    HC->>SvcC: Chọn server thay thế theo tiêu chí cân bằng (active_room_count thấp nhất)
    SvcC->>LeaseStore: Acquire ROOM_LEASE mới (room_id=R1, server=C, TTL=10s)
    LeaseStore-->>SvcC: Lease granted (chỉ granted vì lease cũ của B đã expired, không có 2 lease active song song)
    SvcC->>SnapStore: Đọc ROOM_SNAPSHOT mới nhất của room R1
    SnapStore-->>SvcC: Snapshot version=42, state_blob
    SvcC->>SvcC: Khôi phục state trận đấu từ snapshot version 42
    SvcC->>LB: Cập nhật routing, room R1 giờ resolve về SERVER_INSTANCE C
    LB-->>Players: Reconnect tự động tới SERVER_INSTANCE C
    SvcC-->>Players: Trận đấu tiếp tục từ state snapshot gần nhất, không im lặng drop kết nối
    Note over SvcC,Players: Nếu không tìm thấy snapshot hợp lệ (vd crash quá sớm), chiến lược fallback là kết thúc trận với kết quả hòa thay vì để trạng thái mơ hồ
```
