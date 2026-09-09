# Base ERD — Cụm game server trước khi có consensus

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái **trước khi** cụm có cơ chế đồng thuận (consensus). Base đã có sẵn cụm nhiều node, mỗi trận đấu được gán cho đúng 1 node giữ vai trò "authoritative" (nguồn sự thật cho trạng thái trận), các node còn lại chỉ nhận bản sao trạng thái theo kiểu replicate bất đồng bộ (async, không có log thứ tự, không có xác nhận majority). Input người chơi được xếp thứ tự dựa trên `client_timestamp` do client tự gửi kèm — đây chính là điểm yếu mà enhance phải sửa. Chưa có khái niệm log đồng thuận, term, leader election hay xử lý partition.

```mermaid
erDiagram
    NODE ||--o{ MATCH : "hosts as authoritative"
    MATCH ||--|| MATCH_STATE : "has current"
    MATCH ||--o{ MATCH_STATE_REPLICA : "replicated async to"
    NODE ||--o{ MATCH_STATE_REPLICA : "stores"
    MATCH ||--o{ INPUT_EVENT : "receives"
    PLAYER ||--o{ INPUT_EVENT : "sends"

    NODE {
        string id PK
        string status
    }
    MATCH {
        string id PK
        string authoritative_node_id FK
        string status
    }
    MATCH_STATE {
        string match_id PK
        string state_json
        datetime updated_at
    }
    MATCH_STATE_REPLICA {
        string id PK
        string match_id FK
        string node_id FK
        string state_json
        datetime updated_at
    }
    PLAYER {
        string id PK
        string name
    }
    INPUT_EVENT {
        string id PK
        string match_id FK
        string player_id FK
        string event_type
        datetime client_timestamp
        datetime received_at
        boolean applied
    }
```
