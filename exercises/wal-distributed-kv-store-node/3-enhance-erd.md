# ERD — Enhance (sau khi có WAL cục bộ per-node)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 4 yêu cầu của đề bài — tự phục hồi từ WAL cục bộ trước khi rejoin, phát hiện WAL corrupt để fallback rebuild-from-replica, phối hợp giữa recovery cục bộ và Raft log, và đo lường thời gian phục hồi. So với base, `CLUSTER_NODE` nay có `recovery_mode` để phân biệt đang replay WAL cục bộ hay đang rebuild từ replica, và `LOCAL_WAL_ENTRY` là nguồn phục hồi ưu tiên trước khi chạm tới `RAFT_LOG_ENTRY`.

```mermaid
erDiagram
    CLUSTER_NODE ||--o{ KEY_VALUE_RECORD : stores
    CLUSTER_NODE ||--o{ RAFT_LOG_ENTRY : "participates in"
    CLUSTER_NODE ||--o{ LOCAL_WAL_ENTRY : "writes locally"
    CLUSTER_NODE ||--o| NODE_RECOVERY_STATE : "tracked by"

    CLUSTER_NODE {
        string node_id PK
        string status
        string role
        string recovery_mode
    }

    KEY_VALUE_RECORD {
        string key PK
        string node_id FK
        string value
        datetime updated_at
    }

    RAFT_LOG_ENTRY {
        bigint raft_index PK
        string node_id FK
        string operation
        string key
        string value
        string status
    }

    LOCAL_WAL_ENTRY {
        bigint lsn PK
        string node_id FK
        string operation
        string key
        string value
        string checksum
        string status
        datetime written_at
    }

    NODE_RECOVERY_STATE {
        string state_id PK
        string node_id FK
        string phase
        bigint last_applied_lsn
        bigint last_applied_raft_index
        datetime started_at
        datetime ready_at
    }
```
