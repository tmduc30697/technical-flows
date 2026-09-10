# ERD — Enhance (sau khi có WAL backup cho cache)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 5 yêu cầu của đề bài — ghi WAL trước rồi mới cập nhật cache, replay WAL để rebuild cache khi restart, xử lý đúng thứ tự entry cuối cùng theo từng đơn, cân nhắc sync/batch write với RPO rõ ràng, và phát hiện lệch giữa WAL và cache. So với base, `ORDER_PROCESSING_STATE` trong cache nay luôn bắt nguồn từ `WAL_ENTRY` đã ghi trước đó, không còn là nguồn sự thật duy nhất.

```mermaid
erDiagram
    ORDER ||--o| ORDER_PROCESSING_STATE : "current state in cache"
    ORDER ||--o{ WAL_ENTRY : "state changes logged as"
    ORDER_PROCESSING_STATE ||--o| CONSISTENCY_CHECK : "verified by"

    ORDER {
        string order_id PK
        string customer_id
        decimal total_amount
        datetime created_at
    }

    ORDER_PROCESSING_STATE {
        string order_id PK
        string current_step
        string status
        bigint last_applied_lsn
        datetime updated_at
    }

    WAL_ENTRY {
        bigint lsn PK
        string order_id FK
        string step
        string status
        string write_mode
        datetime written_at
    }

    CONSISTENCY_CHECK {
        string check_id PK
        string order_id FK
        string wal_derived_checksum
        string cache_checksum
        boolean matched
        datetime checked_at
    }
```
