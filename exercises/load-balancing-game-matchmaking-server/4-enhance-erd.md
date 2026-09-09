# Enhance ERD — Consistent hashing theo room, lease TTL, snapshot phục hồi

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, thêm mới:

- `SERVER_INSTANCE.active_room_count` và `virtual_node_count` — đáp ứng yêu cầu 4 (cân bằng tải theo số room active, có trọng số qua virtual node trên hash ring chứ không đếm connection).
- `ROOM.room_hash_slot` — vị trí room trên hash ring, dùng để mọi player join sau đều tra cứu ra đúng server, đáp ứng yêu cầu 1.
- Entity mới `ROOM_LEASE` (room_id, server_instance_id, lease_token, expires_at, status) — cơ chế ownership có TTL để chống split-brain, đáp ứng yêu cầu 3.
- Entity mới `ROOM_SNAPSHOT` (state_blob, version, saved_at) — lưu state gần nhất của room để phục hồi khi server crash giữa trận, đáp ứng yêu cầu 2.
- Entity mới `ROOM_MIGRATION_EVENT` — ghi lại mỗi lần room bị di chuyển giữa các server (do crash hoặc rebalance khi scale cluster), phục vụ đo đạc ở yêu cầu 5.

```mermaid
erDiagram
    SERVER_INSTANCE ||--o{ ROOM : "hiện đang giữ (qua lease active)"
    SERVER_INSTANCE ||--o{ PLAYER : "đang phục vụ connection"
    ROOM ||--o{ PLAYER : "chứa"
    ROOM ||--o{ ROOM_LEASE : "có lịch sử lease"
    ROOM ||--o{ ROOM_SNAPSHOT : "có các bản snapshot"
    ROOM ||--o{ ROOM_MIGRATION_EVENT : "có lịch sử di chuyển"

    SERVER_INSTANCE {
        string id PK
        string host
        int port
        string status "active|draining|down"
        int active_room_count
        int virtual_node_count "trọng số trên hash ring"
    }
    ROOM {
        string id PK
        int room_hash_slot "vị trí trên consistent hash ring"
        string status "active|ended|migrating|draw"
        datetime created_at
    }
    ROOM_LEASE {
        string id PK
        string room_id FK
        string server_instance_id FK
        string lease_token
        datetime acquired_at
        datetime expires_at "TTL ngắn, phải renew định kỳ"
        string status "active|expired|released"
    }
    ROOM_SNAPSHOT {
        string id PK
        string room_id FK
        string server_instance_id FK "server đã tạo snapshot"
        int version
        string state_blob
        datetime saved_at
    }
    ROOM_MIGRATION_EVENT {
        string id PK
        string room_id FK
        string from_server_id
        string to_server_id
        string reason "crash_recovery|scale_up|scale_down"
        bool player_disconnected
        datetime started_at
        datetime completed_at
    }
    PLAYER {
        string id PK
        string room_id FK
        string connected_server_id FK
        datetime connected_at
    }
```
