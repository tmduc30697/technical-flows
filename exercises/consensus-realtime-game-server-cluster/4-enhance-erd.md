# Enhance ERD — Cụm game server có consensus (log + majority)

Đây là **enhance**: mô hình dữ liệu sau khi thêm cơ chế đồng thuận. So với base, các entity mới: `CONSENSUS_LOG` (log sự kiện có `term`/`index`, được replicate và commit theo majority, thay cho việc replicate async không thứ tự), `NODE` được bổ sung `role` (leader/follower/candidate) và `current_term` để phục vụ leader election, `MATCH` được bổ sung `consensus_group` (danh sách node tham gia đồng thuận cho trận đó) và `majority_size`, `TENTATIVE_STATE` ghi nhận trạng thái áp dụng lạc quan chưa được xác nhận (phục vụ trade-off chờ majority và rollback), và `PARTITION_STATUS` ghi nhận trận nào đang ở phía minority cần migrate hoặc pause. `INPUT_EVENT` được bổ sung `idempotency_key` để chống double-apply khi client gửi lại input lúc failover.

```mermaid
erDiagram
    NODE ||--o{ MATCH : "hosts as leader"
    MATCH ||--o{ INPUT_EVENT : "receives"
    PLAYER ||--o{ INPUT_EVENT : "sends"
    MATCH ||--o{ CONSENSUS_LOG : "appends to"
    NODE ||--o{ CONSENSUS_LOG : "written as leader of term"
    CONSENSUS_LOG }o--|| INPUT_EVENT : "orders"
    MATCH ||--|| MATCH_STATE : "has current"
    MATCH ||--o{ TENTATIVE_STATE : "may have pending"
    MATCH ||--o| PARTITION_STATUS : "may be affected by"

    NODE {
        string id PK
        string role
        int current_term
        string status
    }
    MATCH {
        string id PK
        string leader_node_id FK
        string consensus_group_json
        int majority_size
        string status
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
        string idempotency_key
        datetime received_at
    }
    CONSENSUS_LOG {
        string id PK
        string match_id FK
        int term
        int log_index
        string event_id FK
        string leader_node_id FK
        boolean committed
        boolean applied
    }
    MATCH_STATE {
        string match_id PK
        string state_json
        int last_committed_index
        datetime updated_at
    }
    TENTATIVE_STATE {
        string id PK
        string match_id FK
        string event_id FK
        boolean applied_optimistically
        boolean confirmed
    }
    PARTITION_STATUS {
        string id PK
        string match_id FK
        string partition_side
        string action
    }
```
