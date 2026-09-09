# Enhance ERD — Session store với virtual node và replication

Đây là **enhance**: mô hình dữ liệu sau khi thêm virtual node và replication. So với base, các thay đổi/entity mới: `HASH_RING`/`VIRTUAL_NODE` cho phép mỗi node vật lý có nhiều điểm ảo trên ring để tải phân bổ đều hơn (yêu cầu 1); `SESSION` không còn gắn cứng vào 1 `node_id` mà có `SESSION_REPLICA` trên N node kế cận trên ring, mỗi bản sao giữ đúng `expires_at` gốc, không bị reset TTL khi di chuyển (yêu cầu 3, 4); `SESSION_MIGRATION` là bảng "đang di chuyển" để router biết session nào đang trong giai đoạn chuyển tiếp và nên hỏi node nào, tránh session-not-found tạm thời trong lúc rebalance (yêu cầu 2).

```mermaid
erDiagram
    HASH_RING ||--o{ VIRTUAL_NODE : "consists of"
    VIRTUAL_NODE }o--|| NODE : "maps to"
    USER ||--|| SESSION : "has active"
    SESSION ||--o{ SESSION_REPLICA : "replicated to N nodes"
    NODE ||--o{ SESSION_REPLICA : "stores"
    SESSION ||--o| SESSION_MIGRATION : "may be migrating"

    HASH_RING {
        string id PK
        int virtual_nodes_per_physical
    }
    VIRTUAL_NODE {
        string id PK
        string ring_id FK
        string node_id FK
        long hash_point
    }
    NODE {
        string id PK
        string status
    }
    USER {
        string id PK
    }
    SESSION {
        string id PK
        string user_id FK
        string data_json
        datetime created_at
        int ttl_seconds
        datetime expires_at
    }
    SESSION_REPLICA {
        string id PK
        string session_id FK
        string node_id FK
        string role
        string data_json
        datetime expires_at
    }
    SESSION_MIGRATION {
        string id PK
        string session_id FK
        string from_node_id FK
        string to_node_id FK
        string status
        boolean moving
    }
```
