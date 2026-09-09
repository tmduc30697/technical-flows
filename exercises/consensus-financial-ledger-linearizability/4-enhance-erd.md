# Enhance ERD — sau khi có Raft cluster cho ledger

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base (1 node, apply ngay, không idempotency), enhance thêm 4 nhóm entity mới, ứng trực tiếp với các yêu cầu trong đề bài:

- `LEDGER_NODE` (mới) — cluster nhiều node, mỗi node có `role`, `current_term`, `quorum_side` — nền tảng cho yêu cầu 1, 2, 3.
- `TRANSACTION_LOG_ENTRY` (mới) — mỗi giao dịch trước tiên là 1 log entry với `committed` (chỉ true sau majority), `TRANSACTION`/`ACCOUNT` chỉ được apply sau khi entry này commit — đáp ứng yêu cầu 1.
- `TRANSACTION` thêm `transaction_id` duy nhất do client sinh ra, dùng để chống double-apply khi retry — đáp ứng yêu cầu 4.
- `SPLIT_BRAIN_LOG` (mới) — ghi nhận mỗi lần phát hiện 2 leader cùng tồn tại do term cũ và việc tự vô hiệu hoá leader cũ — đáp ứng yêu cầu 3.
- `LEDGER_METRIC`/`QUORUM_ALERT` (mới) — commit latency p50/p99, leader election count 24h, alert mất quorum — đáp ứng yêu cầu đo lường.

```mermaid
erDiagram
    ACCOUNT ||--o{ TRANSACTION : "debited/credited by"
    LEDGER_NODE ||--o{ TRANSACTION_LOG_ENTRY : "hosts local copy of"
    TRANSACTION_LOG_ENTRY ||--o| TRANSACTION : "applies to state machine after commit"
    LEDGER_NODE ||--o{ SPLIT_BRAIN_LOG : "may trigger"
    LEDGER_NODE ||--o{ LEDGER_METRIC : "reports"
    LEDGER_NODE ||--o{ QUORUM_ALERT : "raises when minority too long"

    ACCOUNT {
        string id PK
        string owner_name
        decimal balance
    }
    TRANSACTION {
        string id PK
        string transaction_id UK "client-generated, duy nhất"
        string from_account_id FK
        string to_account_id FK
        decimal amount
        datetime applied_at
    }
    LEDGER_NODE {
        string id PK
        string role "leader | follower | disabled_old_leader"
        int current_term
        string quorum_side "majority | minority | unknown"
    }
    TRANSACTION_LOG_ENTRY {
        string id PK
        int index
        int term
        string node_id FK
        string transaction_id FK
        int replicated_count
        boolean committed
        datetime appended_at
    }
    SPLIT_BRAIN_LOG {
        string id PK
        string old_leader_node_id FK
        int old_term
        int new_term_discovered
        datetime detected_at
        datetime old_leader_disabled_at
    }
    LEDGER_METRIC {
        string id PK
        string metric_name "commit_latency_p50_ms | commit_latency_p99_ms | leader_election_count_24h"
        float value
        datetime recorded_at
    }
    QUORUM_ALERT {
        string id PK
        int seconds_without_quorum
        boolean alert_triggered
        datetime detected_at
    }
```
