# Enhance ERD — Rate limiter cluster với consistent hashing dùng chung

Đây là **enhance**: mô hình dữ liệu sau khi mọi router instance tra chung 1 `HASH_RING` duy nhất thay vì giữ bảng cục bộ riêng. So với base, các entity mới: `HASH_RING`/`VIRTUAL_NODE` là nguồn sự thật duy nhất cho việc route key sang node (thay `ROUTING_TABLE_ENTRY` cục bộ, giải quyết yêu cầu 1); `COUNTER_MIGRATION` theo dõi việc chuyển giá trị counter hiện tại từ node cũ sang node mới khi rebalance, có cờ `locked` để chặn race condition khi cả node cũ và mới cùng nhận request cho key đang di chuyển (yêu cầu 2, 3); `HOT_KEY` và `HOT_KEY_PARTIAL_COUNTER` cho phép chia nhỏ việc đếm 1 key traffic cực lớn ra nhiều node rồi tổng hợp định kỳ (yêu cầu 4).

```mermaid
erDiagram
    HASH_RING ||--o{ VIRTUAL_NODE : "consists of"
    VIRTUAL_NODE }o--|| NODE : "maps to"
    ROUTER_INSTANCE }o--|| HASH_RING : "queries shared"
    NODE ||--o{ RATE_LIMIT_COUNTER : "holds"
    API_KEY ||--o{ RATE_LIMIT_COUNTER : "counted in"
    API_KEY ||--o| COUNTER_MIGRATION : "may be migrating"
    API_KEY ||--o| HOT_KEY : "may be flagged as"
    HOT_KEY ||--o{ HOT_KEY_PARTIAL_COUNTER : "split across nodes"

    ROUTER_INSTANCE {
        string id PK
    }
    HASH_RING {
        string id PK
        int total_virtual_nodes
        string ring_version
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
    API_KEY {
        string id PK
        string owner
    }
    RATE_LIMIT_COUNTER {
        string id PK
        string api_key_id FK
        string node_id FK
        datetime window_start
        int count
        int limit_per_window
    }
    COUNTER_MIGRATION {
        string id PK
        string api_key_id FK
        string from_node_id FK
        string to_node_id FK
        int transferred_count
        boolean locked
        string status
    }
    HOT_KEY {
        string id PK
        string api_key_id FK
        int detected_qps
        datetime detected_at
    }
    HOT_KEY_PARTIAL_COUNTER {
        string id PK
        string hot_key_id FK
        string node_id FK
        int partial_count
        datetime last_synced_at
    }
```
