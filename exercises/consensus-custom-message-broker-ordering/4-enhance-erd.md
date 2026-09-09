# Enhance ERD — sau khi có consensus giữa broker node theo partition

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base (1 broker/partition, offset commit gắn với broker đó), enhance thêm nhóm entity mô hình hoá replica group + consensus, ứng trực tiếp với các yêu cầu trong đề bài:

- `BROKER_NODE` (mới) và `PARTITION_REPLICA_GROUP` (mới) — mỗi partition nay có nhiều broker node (leader + follower), thay vì 1 broker duy nhất.
- `MESSAGE` thêm `replicated_count`/`committed` — offset chỉ chính thức khi replicate tới majority — đáp ứng yêu cầu 1.
- `LEADER_TERM` (mới) — mỗi lần đổi leader tăng term, phục vụ xác định leader mới xác định đúng last committed offset — đáp ứng yêu cầu 2.
- `CONSUMER_OFFSET` nay thuộc `CONSUMER_OFFSET_STORE` (mới) độc lập với broker leader — đáp ứng yêu cầu 4.
- `FAILOVER_METRIC` (mới) — failover time, tần suất đổi leader, kết quả chaos test — đáp ứng yêu cầu đo lường.

```mermaid
erDiagram
    TOPIC ||--o{ PARTITION : "divided into"
    PARTITION ||--|| PARTITION_REPLICA_GROUP : "backed by"
    PARTITION_REPLICA_GROUP ||--o{ BROKER_NODE : "consists of"
    PARTITION_REPLICA_GROUP ||--o{ LEADER_TERM : "has history of"
    PARTITION ||--o{ MESSAGE : "stores in order"
    CONSUMER_GROUP ||--o{ CONSUMER_OFFSET_STORE : "tracks per partition"
    PARTITION ||--o{ CONSUMER_OFFSET_STORE : "read progress of"
    PARTITION_REPLICA_GROUP ||--o{ FAILOVER_METRIC : "reports"

    TOPIC {
        string id PK
        string name
    }
    PARTITION {
        string id PK
        string topic_id FK
    }
    PARTITION_REPLICA_GROUP {
        string id PK
        string partition_id FK
        string current_leader_broker_id
        int current_term
    }
    BROKER_NODE {
        string id PK
        string replica_group_id FK
        string role "leader | follower"
        int last_known_committed_offset
    }
    LEADER_TERM {
        string id PK
        string replica_group_id FK
        int term
        string leader_broker_id
        datetime started_at
        datetime ended_at
    }
    MESSAGE {
        string id PK
        string partition_id FK
        int offset
        string payload
        int replicated_count
        boolean committed
        datetime published_at
    }
    CONSUMER_GROUP {
        string id PK
        string name
    }
    CONSUMER_OFFSET_STORE {
        string id PK
        string consumer_group_id FK
        string partition_id FK
        int committed_offset
        string storage_location "độc lập, không nằm trên broker leader"
    }
    FAILOVER_METRIC {
        string id PK
        string replica_group_id FK
        string metric_name "failover_time_ms | leader_change_count | chaos_test_result"
        float value
        datetime recorded_at
    }
```
