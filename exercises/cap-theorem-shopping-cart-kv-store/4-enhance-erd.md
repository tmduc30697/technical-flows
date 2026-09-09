# Enhance ERD — sau khi có tunable consistency và merge rõ ràng

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm entity mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `QUORUM_CONFIG` thêm `satisfies_wr_gt_n`/`rationale` — lý giải rõ tại sao chọn W+R>N hay không.
- `PARTITION_WRITE_POLICY` (mới) — ưu tiên availability cho "thêm vào giỏ", chấp nhận ghi phía minority.
- `CART_CONFLICT_LOG` (mới) — chiến lược merge union khi partition hàn lại, đồng thời đo tần suất conflict.
- `READ_CONSISTENCY_LEVEL` (mới) — client chọn mức consistency theo use case (checkout vs header icon).

```mermaid
erDiagram
    CART ||--o{ CART_REPLICA : "replicated as"
    CART ||--o{ CART_CONFLICT_LOG : "may generate"
    QUORUM_CONFIG ||--|| PARTITION_WRITE_POLICY : "paired with"
    QUORUM_CONFIG ||--o{ READ_CONSISTENCY_LEVEL : "defines per use case"

    CART {
        string id PK
        string user_id
    }
    CART_REPLICA {
        string id PK
        string cart_id FK
        string node_id
        string items_json
        datetime updated_at
    }
    QUORUM_CONFIG {
        string id PK
        int n_replica
        int write_quorum
        int read_quorum
        boolean satisfies_wr_gt_n
        string rationale
    }
    PARTITION_WRITE_POLICY {
        string id PK
        string action "add_to_cart"
        boolean accept_minority_writes
    }
    CART_CONFLICT_LOG {
        string id PK
        string cart_id FK
        string side_a_items
        string side_b_items
        string merged_items
        string resolution "union_no_delete"
        datetime resolved_at
    }
    READ_CONSISTENCY_LEVEL {
        string id PK
        string use_case "checkout | header_icon"
        int read_quorum
    }
```
