# Enhance sequence — Test scale cluster 3 → 6 instance, đo rebalance không disconnect

Đây là **enhance**, flow test mô phỏng mô tả ở yêu cầu 5 của đề bài: scale cluster từ 3 lên 6 instance khi đang có hàng trăm room active, đo số room bị di chuyển (rebalance) và xác nhận player không bị disconnect. Test này cũng xác nhận việc cân bằng tải theo số room active (yêu cầu 4) hoạt động đúng khi thêm instance mới.

```mermaid
sequenceDiagram
    participant TestRunner as Test Runner
    participant LB as Load Balancer / Hash Ring
    participant Cluster3 as Cluster (3 instances, ~300 room active)
    participant NewNodes as 3 SERVER_INSTANCE mới (scale lên 6)
    participant MigLog as ROOM_MIGRATION_EVENT log
    actor Players as Players đang chơi

    TestRunner->>Cluster3: Khởi tạo baseline, hàng trăm room active, ghi nhận room_count mỗi server
    TestRunner->>LB: Thêm 3 SERVER_INSTANCE mới vào hash ring (thêm virtual nodes)
    LB->>LB: Tính lại phân bố slot trên ring với 6 instance
    LB->>NewNodes: Xác định các room cần chuyển sang instance mới để cân bằng active_room_count
    loop Với mỗi room cần di chuyển
        LB->>Cluster3: Yêu cầu server cũ tạo ROOM_SNAPSHOT trước khi giải phóng lease
        Cluster3->>MigLog: Ghi ROOM_MIGRATION_EVENT (reason=scale_up, started_at)
        Cluster3->>NewNodes: Chuyển lease + snapshot sang server mới
        NewNodes->>Players: Reconnect qua LB, tiếp tục trận từ snapshot, không mất kết nối
        NewNodes->>MigLog: Cập nhật ROOM_MIGRATION_EVENT (completed_at, player_disconnected=false)
    end
    TestRunner->>MigLog: Truy vấn tổng số room đã migrate và player_disconnected
    MigLog-->>TestRunner: Vd 120/300 room migrated (chỉ số room cần để cân bằng lại theo active_room_count), player_disconnected=0
    TestRunner-->>TestRunner: Assert số room migrate hợp lý theo thuật toán consistent hashing (không phải toàn bộ room bị xáo trộn), và player_disconnected=0 cho toàn bộ test
```
