# Enhance ERD — sau khi tách consistency counter/relationship

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm thay đổi chính, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `QUORUM_POLICY` (mới) — tách riêng quorum theo mục đích: relationship write/read dùng quorum cao, counter read dùng quorum thấp.
- `FOLLOW_RELATIONSHIP` thêm `version`/`last_action_timestamp` — xử lý đúng thứ tự khi follow/unfollow gần như đồng thời.
- `PARTITION_WRITE_POLICY` (mới) — ưu tiên availability cho follow lúc partition, merge theo "follow thắng nếu không xác định được".
- `FOLLOWER_COUNT` chuyển sang cập nhật bất đồng bộ qua `COUNTER_UPDATE_JOB`; `COUNTER_RECONCILIATION_REPORT` (mới) — đối soát định kỳ với ground truth và tự điều chỉnh khi lệch vượt ngưỡng.

```mermaid
erDiagram
    FOLLOW_RELATIONSHIP ||--o{ RELATIONSHIP_REPLICA : "replicated as"
    FOLLOW_RELATIONSHIP ||--o{ COUNTER_UPDATE_JOB : "queues delta for"
    FOLLOWER_COUNT ||--o{ COUNTER_UPDATE_JOB : "applied by"
    FOLLOWER_COUNT ||--o{ COUNTER_RECONCILIATION_REPORT : "audited via"

    FOLLOW_RELATIONSHIP {
        string id PK
        string follower_id
        string followee_id
        string status "following | not_following"
        int version
        datetime last_action_timestamp
    }
    RELATIONSHIP_REPLICA {
        string id PK
        string relationship_id FK
        string node_id
        string status
        int version
        datetime updated_at
    }
    QUORUM_POLICY {
        string id PK
        string purpose "relationship_write | relationship_read | counter_read"
        int quorum_value
    }
    PARTITION_WRITE_POLICY {
        string id PK
        string action "follow"
        boolean accept_minority_writes
        string conflict_resolution "follow_wins_if_ambiguous"
    }
    FOLLOWER_COUNT {
        string user_id PK
        int approximate_count
        datetime last_batch_applied_at
    }
    COUNTER_UPDATE_JOB {
        string id PK
        string user_id FK
        int batch_delta
        datetime scheduled_at
        datetime applied_at
    }
    COUNTER_RECONCILIATION_REPORT {
        string id PK
        string user_id FK
        int approximate_count
        int ground_truth_count
        int drift
        boolean auto_corrected
        datetime generated_at
    }
```
