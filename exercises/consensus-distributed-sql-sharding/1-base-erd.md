# Base ERD — Distributed SQL DB trước khi chia range/multi-raft

Đây là **base**: mô hình dữ liệu suy luận trước khi áp sharding theo range. Đề bài mô tả vai trò flow là "mỗi range chạy Raft riêng để đồng thuận... multi-raft trên cùng cluster" — nghĩa là trước đó toàn bộ dữ liệu của bảng nằm trong đúng 1 range duy nhất, chạy đúng 1 Raft group duy nhất cho cả bảng, chưa có khái niệm chia nhỏ theo range hay client phải tìm đúng leader của từng range (vì chỉ có 1 leader duy nhất cho toàn bộ dữ liệu).

```mermaid
erDiagram
    TABLE ||--|| RANGE_DATA : "stored entirely in one"
    RANGE_DATA ||--|| RAFT_GROUP : "backed by exactly one"
    RAFT_GROUP ||--o{ REPLICA_NODE : "consists of"

    TABLE {
        string id PK
        string name
    }
    RANGE_DATA {
        string id PK
        string table_id FK
        string key_start "toàn bộ key space"
        string key_end
    }
    RAFT_GROUP {
        string id PK
        string range_id FK
        string leader_node_id
        int current_term
    }
    REPLICA_NODE {
        string id PK
        string raft_group_id FK
        string role "leader | follower"
    }
```
