# Base ERD — Matchmaking server trước khi có consistent hashing theo room

Đây là **base**: trạng thái backend game multiplayer *trước khi* áp dụng consistent hashing theo `room_id`. Suy luận từ đề bài, base đã có `SERVER_INSTANCE` (các game server trong cluster), `ROOM` (được gán vào 1 server khi tạo, nhưng gán theo cách đơn giản như round-robin/random dựa trên số connection), và `PLAYER` kết nối vào 1 server cụ thể. Base chưa có khái niệm lease/ownership có TTL, chưa có snapshot phục hồi state, và load balancing chỉ dựa trên số connection chứ chưa tính theo số room active.

```mermaid
erDiagram
    SERVER_INSTANCE ||--o{ ROOM : "được gán giữ"
    SERVER_INSTANCE ||--o{ PLAYER : "đang phục vụ connection"
    ROOM ||--o{ PLAYER : "chứa"

    SERVER_INSTANCE {
        string id PK
        string host
        int port
        string status "active|down"
        int active_connection_count
    }
    ROOM {
        string id PK
        string assigned_server_id FK
        string status "active|ended"
        datetime created_at
    }
    PLAYER {
        string id PK
        string room_id FK
        string connected_server_id FK
        datetime connected_at
    }
```
