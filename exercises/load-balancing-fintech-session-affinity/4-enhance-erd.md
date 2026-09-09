# Enhance ERD — sau khi có consistent hashing, virtual node, failover và chống timing attack

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có các entity/field mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `HASH_RING` + `VIRTUAL_NODE` (mới) — vòng hash với nhiều điểm ảo cho mỗi instance, tránh phân bố lệch tải khi cluster nhỏ (yêu cầu 1, 4).
- `TRANSACTION` (sửa) — thêm `hash_value`, `assigned_instance_id` để cùng `transaction_id` luôn map về đúng một instance trong suốt vòng đời giao dịch (yêu cầu 1).
- `TRANSACTION_STATE_SNAPSHOT` (mới) — state được ghi bền vững (DB/cache) sau mỗi bước thay vì chỉ giữ trong bộ nhớ, làm nguồn khôi phục khi failover (yêu cầu 2).
- `INSTANCE_FAILOVER_EVENT` (mới) — ghi nhận việc phát hiện instance chết giữa giao dịch và chuyển sang instance khác kèm khôi phục state (yêu cầu 2).
- `RING_REBALANCE_EVENT` (mới) — ghi nhận mỗi lần thêm/bớt instance khỏi ring, kèm số liệu % key bị remap để verify đặc tính consistent hashing (yêu cầu 3).
- `ROUTING_TIMING_SAMPLE` (mới) — đo thời gian route của từng request để đảm bảo không có sự khác biệt đáng kể giữa các user, tránh lộ thông tin qua timing attack (yêu cầu 5).

```mermaid
erDiagram
    HASH_RING ||--o{ VIRTUAL_NODE : "gồm nhiều điểm ảo"
    VIRTUAL_NODE }o--|| INSTANCE : "thuộc về"
    TRANSACTION }o--|| INSTANCE : "được gán cố định"
    TRANSACTION ||--o{ TRANSACTION_STEP : "có nhiều bước"
    TRANSACTION_STEP }o--|| INSTANCE : "thực thi tại"
    TRANSACTION ||--|| TRANSACTION_STATE_SNAPSHOT : "có state bền vững"
    TRANSACTION ||--o{ INSTANCE_FAILOVER_EVENT : "có thể gặp"
    HASH_RING ||--o{ RING_REBALANCE_EVENT : "ghi nhận thay đổi ring"
    TRANSACTION ||--o{ ROUTING_TIMING_SAMPLE : "được đo thời gian route"

    HASH_RING {
        string id PK
        string algorithm "consistent_hashing_hmac_sha256"
        int virtual_nodes_per_instance
        datetime updated_at
    }
    VIRTUAL_NODE {
        string id PK
        string hash_ring_id FK
        string instance_id FK
        int virtual_index
        bigint hash_position
    }
    INSTANCE {
        string id PK
        string host
        string status "up | down"
    }
    TRANSACTION {
        string id PK
        string user_id
        string transaction_type "transfer | otp_verify"
        string status "pending | completed | failed"
        bigint hash_value
        string assigned_instance_id FK
        datetime created_at
    }
    TRANSACTION_STEP {
        string id PK
        string transaction_id FK
        int step_number
        string instance_id FK
        datetime executed_at
    }
    TRANSACTION_STATE_SNAPSHOT {
        string transaction_id PK, FK
        string state_payload
        int version
        datetime updated_at
    }
    INSTANCE_FAILOVER_EVENT {
        string id PK
        string transaction_id FK
        string dead_instance_id FK
        string new_instance_id FK
        datetime detected_at
        datetime state_restored_at
    }
    RING_REBALANCE_EVENT {
        string id PK
        string triggered_by "scale_out | scale_in"
        string instance_ref_id FK
        int keys_remapped_count
        int keys_total_count
        decimal remap_percentage
        datetime occurred_at
    }
    ROUTING_TIMING_SAMPLE {
        string id PK
        string transaction_id FK
        int routing_duration_ms
        datetime recorded_at
    }
```
