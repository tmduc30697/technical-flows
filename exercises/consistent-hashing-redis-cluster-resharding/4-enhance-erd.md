# Enhance ERD — Cache cluster với consistent hashing và migration job

Đây là **enhance**: mô hình dữ liệu sau khi cụm dùng consistent hashing với virtual node. So với base, các entity mới: `HASH_RING`/`VIRTUAL_NODE` — mỗi node vật lý có nhiều điểm ảo trên ring để tránh hot node khi số node vật lý ít (yêu cầu 2), thay cho hashing modulo N cứng; `MIGRATION_JOB` theo dõi tiến trình resharding cho 1 dải key cụ thể, có `dual_write_active` để xử lý đọc/ghi đúng trong lúc key đang di chuyển (yêu cầu 3), và `checkpoint_key`/`rollback_available` để có thể rollback giữa chừng mà không mất phần đã migrate xong (yêu cầu 4).

```mermaid
erDiagram
    HASH_RING ||--o{ VIRTUAL_NODE : "consists of"
    VIRTUAL_NODE }o--|| NODE : "maps to"
    NODE ||--o{ CACHE_ENTRY : "stores"
    HASH_RING ||--o{ MIGRATION_JOB : "tracks resharding via"
    MIGRATION_JOB }o--|| NODE : "from node"
    MIGRATION_JOB }o--|| NODE : "to node"

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
        string host
        string status
    }
    CACHE_ENTRY {
        string key PK
        string node_id FK
        string value
        datetime updated_at
    }
    MIGRATION_JOB {
        string id PK
        string key_range_start
        string key_range_end
        string from_node_id FK
        string to_node_id FK
        string status
        boolean dual_write_active
        string checkpoint_key
        boolean rollback_available
    }
```
