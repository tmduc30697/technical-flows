# ERD — Enhance (sau khi có WAL cho message queue)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 4 yêu cầu của đề bài — chỉ ack producer sau khi WAL fsync, replay đúng thứ tự + trạng thái ack sau crash, redeliver message đang xử lý dở khi crash (đòi hỏi consumer idempotent), và checkpoint định kỳ cắt bớt WAL. So với base, mọi thay đổi trạng thái của `MESSAGE` (nhận, giao cho consumer, ack) đều được ghi thành `WAL_ENTRY` riêng biệt, không chỉ ghi lúc nhận như base.

```mermaid
erDiagram
    PRODUCER ||--o{ MESSAGE : sends
    QUEUE ||--o{ MESSAGE : holds
    CONSUMER ||--o{ MESSAGE : "reads/acks"
    MESSAGE ||--|{ WAL_ENTRY : "state changes logged as"
    CHECKPOINT ||--o{ WAL_ENTRY : "covers up to"

    PRODUCER {
        string producer_id PK
        string name
    }

    QUEUE {
        string queue_id PK
        string name
    }

    MESSAGE {
        string message_id PK
        string queue_id FK
        string producer_id FK
        string payload
        string ack_status
        datetime received_at
    }

    CONSUMER {
        string consumer_id PK
        string name
    }

    WAL_ENTRY {
        bigint lsn PK
        string message_id FK
        string event_type
        string consumer_id FK
        datetime written_at
    }

    CHECKPOINT {
        bigint checkpoint_id PK
        bigint up_to_lsn
        datetime created_at
    }
```
